---
title: "RubyのRactor::Portを用いてCICDパイプラインランナーをスクラッチで実装する"
date: 2025-11-16T17:08:05+09:00
draft: false
---

先日、HackerNewsで面白い記事を見つけた。  
[Building a CI/CD Pipeline Runner from Scratch in Python | Muhammad](https://muhammadraza.me/2025/building-cicd-pipeline-runner-python/)  
ちょうど、依存関係にある処理のオーケストレーションに興味があったので[^1]、Rubyで同様のCICDパイプラインランナーをスクラッチで実装してみることにした。


## パイプラインランナー
前述の記事に記載があるが、今回構築するパイプラインランナーは以下の機能を持つ。

- ymlをパースする
- ジョブの依存関係を解決する
- ジョブを実行する（依存に問題がなければ並列実行も可能）
- ログをストリームする
- ジョブ間でアーティファクトを共有する
- 結果を報告する

今回は並列実行にRubyのRactorを使用している。サンプルコードはGitHubに乗せている。   
`ruby ci_cd.rb pipeline.yml`でどのバージョンも実行できる。サンプルコードはRuby 4.0-devで登場した機能を利用しており、それ以前のバージョンだと動作しないので注意。

 ## Version 1: 単一のジョブを実行する

まずは、単一のジョブを実行するだけのシンプルなバージョンを作成した。
ymlを読んで、ジョブを実行し、ログを表示するだけのものだ。

[version 1 Single Job Executor](https://github.com/QWYNG/Building-CICD-Pipeline-Runner-Ruby-from-Scratch/commit/c6ebd0db1b2c3757765131f2dd57be17c6665d74)

Rubyの`Open3.popen2e`は標準出力と標準エラー出力を同時に扱えて便利。僕が知らないだけでPythonにも同様の機能があるのかもしれないが。

## Version 2: 複数のステージを逐次実行する
次に、複数のステージをそれぞれ逐次実行するバージョンを作成した。

[Version 2: Multi-Stage Pipeline with Sequential Execution](https://github.com/QWYNG/Building-CICD-Pipeline-Runner-Ruby-from-Scratch/commit/da0b8bf37616faba313f5a009f1a7c29715da205)

## Version 3: ジョブを並列実行する
ここが今回のハイライト。参考ブログではPythonの`multiprocessing`を使用していた。この記事ではRubyのRactorを使用する。  
[Version 3: Parallel Job Execution](https://github.com/QWYNG/Building-CICD-Pipeline-Runner-Ruby-from-Scratch/commit/e805638da91444dd98a2bda73a94548be4a6cabe)


このVersionでは、並列に実行されるジョブの結果を集約し、失敗か成功か確認する処理がある。  
今回はファイル一つだけの簡単なスクリプトなので、メインのRactorに結果を全部送信してしまってもいいが、他のライブラリでメインのRactorに何を送信するかもわからないので、専用の通信路を用意したほうがいいだろう。  
これまでだと、こういった用途を限定した通信路を作成するためにはchannelを用意してRactor間で共有する必要があった。

```ruby
result_channel = Ractor.new do
                   while true
                     Ractor.yield Ractor.receive
                   end
                 end


stage_jobs.map do |job|
  Ractor.new(job, workspace, result_channel) do |j, ws, result_channel|
    result_channel << JobExecutor.new(ws).run(j)
  end
end

results = stage_jobs.size.times.map do
  result_channel.take
end
```

儀礼的にchannel書くの面倒くさいな〜と思っていた時にRactor::Port[^2]。  
Ractor::PortはRubyでこれから導入される予定[^3]の機能で、軽量かつ生成したRactorでしか受け取れない受信箱をつくる機能である。  
今回は各ジョブの結果を受け取る箱として使用した。スッキリして( ・∀・)ｲｲ!!


```ruby
result_port = Ractor::Port.new

stage_jobs.map do |job|
  Ractor.new(job, workspace, result_port) do |j, ws, result_port|
    result_port << JobExecutor.new(ws).run(j)
  end
end

results = stage_jobs.size.times.map do
  result_port.receive
end
```

## Version 4: 依存関係とアーティファクト

依存関係の導入と、アーティファクト生成、他のジョブの残したアーティファクトを利用できる機能を導入した。

[Version 4: Dependencies and Artifacts](https://github.com/QWYNG/Building-CICD-Pipeline-Runner-Ruby-from-Scratch/commit/46dde4b2ef6a058babc8a42ef463037dbbab3503)

ちょっと困ったのが、アーティファクトの概念を導入する際、以下の用に`Pathname#relative_path_from`をnon main ractor内で使用すると`Ractor::IsolationError`が出てしまうということ。

```ruby
relative_path = artifact_path.relative_path_from(job_artifact_dir)
# => 'Pathname#relative_path_from': can not access non-shareable objects in constant Pathname::SAME_PATHS by non-main ractor. (Ractor::IsolationError)
```

[Pathname::SAME_PATHSの実装](https://github.com/ruby/ruby/blob/bacd35626c500db1f27136b57363685e4da74d9b/pathname_builtin.rb#L201)をみると共有可能なProcになっていなかったのでしょうがなさそう。  
今回は「ベースのパスを消せば実質相対パス！」という荒業で回避した。

```ruby
relative_path = artifact.delete_prefix("#{job_artifact_dir}/")
```

依存関係の解決の為にトポロジカルソートも実装したが、これは元のPythonのコードとそれほど変わらない。依存が無くなったものから並列処理できるよう配列にまとめておくのがポイント。

```ruby
  def topological_sort(jobs)
    execution_order = []
    job_map = jobs.map { |job| [job.name, job] }.to_h
    in_degree = jobs.map { |job| [job.name, 0] }.to_h
    adjacency = Hash.new { |h, k| h[k] = [] }

    jobs.each do |job|
      job.needs.each do |dep|
        adjacency[dep] << job.name if job_map.key?(dep)
        in_degree[job.name] += 1 if job_map.key?(dep)
      end
    end

    queue = in_degree.select { |_, degree| degree.zero? }.keys

    while queue.any?
      current = queue
      execution_order << current.map { |job_name| job_map[job_name] }
      queue = []

      current.each do |job_name|
        adjacency[job_name].each do |neighbor|
          in_degree[neighbor] -= 1
          queue << neighbor if in_degree[neighbor].zero?
        end
      end
    end

    raise 'Cyclic dependency detected in job definitions.' if execution_order.flatten.size != jobs.size

    execution_order
  end
```
実際に書いてみて、依存がない順にジョブを足していく感覚を得れたので良かった。

## Version 5: 変数の導入、ブランチの指定
変数を用いた書き換えと、ブランチの指定を実装した。  
[Version 5: Production-Ready Features](https://github.com/QWYNG/Building-CICD-Pipeline-Runner-Ruby-from-Scratch/commit/5aea95f7a12fe4f5b32a983fbc363fef873f07f0)

これはどちらかというとおまけかな。設定ファイルに記述された変数を置換しているだけ。  
ただ、実際にPythonアプリのCICDが走って感動した。

## まとめ
スクリプト書くのたのし〜。  
Ractor::Port、使いやすくていいですね。リリースが楽しみです。

[^1]: overmindというtmuxのセッション起動いい感じにしてくれる君のプロセスの依存に関するissueを解決したかった。(https://github.com/DarthSim/overmind/issues/70)
[^2]: [Ractor::Port ― Ractor の API を一新した話 - STORES Product Blog](https://product.st.inc/entry/2025/06/24/110606)
[^3]: Ruby 3.5.0-preview1ではRactor::Portは使えなかった。筆者は3.4.0-devを利用した。














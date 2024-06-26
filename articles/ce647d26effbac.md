---
title: "グラフアルゴリズムの基礎を学ぶー単一始点最短経路問題"
emoji: "🗺️"
type: "tech"
topics:
  - "algorithm"
  - "graph"
published: true
published_at: "2023-05-06 22:55"
---

この記事は、グラフアルゴリズムシリーズの5番目の記事です。

1. [概念と表現方法](https://zenn.dev/convers39/articles/9b66f263e335d8)
2. [BFSとDFS](https://zenn.dev/convers39/articles/1c315cd96a991f)
3. [UnionFind](https://zenn.dev/convers39/articles/ffd666639e7782)
4. [最小全域木](https://zenn.dev/convers39/articles/6126e22dd116fb)
5. 単一始点最短経路問題→現在地
6. 全点対最短経路問題（WIP）

![](https://storage.googleapis.com/zenn-user-upload/5e7f33f33fe4-20230506.png)

---

単一始点最短経路問題(Single Source Shortest Path Problem)というのは、**ウェイトありグラフにおいて、始点から終点までの最小コストの経路を探索する**問題を指しています。基礎編で触れたBFSも最短経路に適しているのですが、ウェイトなしのグラフしか適応できません。この問題は基本的に、始点から全てのノードまでの最短経路を見つけ出すことが目的になっています。現実生活では、ウェイト付きの場面が多くあるので、それを解決するための2つのアルゴリズムをここで見てみたいと思います。

## Dijkstra's Algorithm

このアルゴリズムを一言で言えば、任意のノードを始点として、目標となるノードに到達するまで、**始点から各ノードまでの合計最小コストを更新しながら経路を拡張していく**ことです。

![](https://storage.googleapis.com/zenn-user-upload/f93180111647-20230506.gif)
出典：https://commons.wikimedia.org/wiki/File:Dijkstra_Animation.gif

アルゴリズムは以下となります。
- 始点ノードを選択し、そのノードへの最短距離を0に設定し、他のすべてのノードへの最短距離を無限大（inf）に初期化します。
- ヒープを作成し、始点ノードをキューに追加します。キューの各アイテムは、**始点からの最短距離**とノードのペアとなります。
- キューが空になるまで以下の操作を繰り返します。
  - キューから最小距離のノードを取り出します。
  - すでに見つかっているより短い経路がある場合は、現在のノードをスキップします。
  - 現在のノードから隣接するすべてのノードに対して、新しい経路が現在の最短距離よりも短いかどうかを確認します。
  - 短ければ、最短距離を更新し、そのノードと新しい距離をキューに追加します。

```python
import heapq

def dijkstra(start, graph):
    V = len(graph)
    dist = [float('inf')] * V
    dist[start] = 0

    pq = []
    heapq.heappush(pq, (0, start))

    while pq:
        cur_dist_from_start, cur_node = heapq.heappop(pq)

        if cur_dist_from_start > dist[cur_node]:
            continue

        for next_node, edge_weight in graph[cur_node]:
            # 現在のノードまでのコストに次のノードまでのウェイトを乗せて、最小値を更新する
            dist_to_next_node = dist[cur_node] + edge_weight
            if dist[next_node] > dist_to_next_node:
                dist[next_node] = dist_to_next_node
                heapq.heappush(pq, (dist_to_next_node, next_node))

    return dist_to
```

上記の実装を見て気づくかもしれませんが、このアルゴリズムは少しプリム法と似ているところがあります。いずれも貪欲法で、ヒープでコストとノードを保存し、BFSで探索しています。違いといえば、プリム法のヒープでは、ノード間のコストを保存しているのに対して、ダイクストラ法では始点からノードまでの最小コスト合計を保存しています。これは、MSTを探し出すことと、始点から各ノードまでの合計最小コストを探し出すことといった目的の違いに由来します。

また、`dist_to`配列や、`for`ループの部分の値更新ロジックを見ると、動的計画法・DP(Dynamic Programming)の既視感はあるのではないかと。実質ダイクストラ法と次に紹介するベルマンフォード法も、DPの考え方を利用しています。

[こちらの動画](https://youtu.be/EFg3u_E6eHU)はかなりわかりやすく説明してくれていますので、イメージが掴めない場合は一見する価値はあると思います。

時間複雑度について、バイナリーヒープで想定する場合、エッジ数の回数分ヒープポップ操作するので、`O(E*logV)`となります。全てのノードを探索するため、さらに`O(V)`がかかりますので、合計で`O(V+ElogV)`となります。空間複雑度について、ノード数に対応する配列やヒープが必要なので、`O(V)`となります。

一つ注意しないといけないのが、このアルゴリズムは、ウェイトが正数の場合のみ効果があります。仮にウェイトがマイナスになると、貪欲法は失敗します。この問題を解決するために、下記のBellman-Ford法が助けになります。

## Bellman-Ford Algorithm

アルゴリズムの説明に入る前に、少しサイクルのことを説明します。

ダイクストラ法はウェイトが正数の場合に有力ですが、マイナスウェイトが存在すると、問題は複雑になります。

グラフのサイクルについて、**巡回図（cyclic graph）**と**非巡回図（acyclic graph）**のように概念として区別しています。サイクルのない図において、2つのノードの間の経路のパターンに限りがありますが、サイクルのある図には、サイクルを繰り返すことで実際に経路のパターンが無限になります。

最小コストの計算においても、このサイクルの有無と、マイナスウェイトの有無に大きく影響されます（ダイクストラ法が使えない）。さらに、マイナスウェイトの存在により、合計コストがマイナスとなるサイクルを生み出す可能性があります。下記の巡回図を考えてみてください。

![](https://storage.googleapis.com/zenn-user-upload/6735474bffa4-20230506.png)

ここで仮にノードAからノードBまでの最小コストの経路を探すこととします。B->Aのウェイトが-4となるので、理論的にサイクルに辿れば、A->C->B->A->C->B->...のように最小コストが無限に小さくなっていきます。このパターンになると、最小コストの結果が見つかりませんが（存在しないため）、ベルマンフォード法は、このパターンを検出することができます。

### DPとベルマンフォード法

ダイクストラ法のところで少し触れましたが、ベルマンフォード法はDPを運用しています。DPをシンプルにいえば、大きな問題を小さく分解し、分解された問題の解を組み合わせて最適解を見つけることです。ベルマンフォード法を一言で言えば、**始点から全てのノードまでの合計最小コストを計算するために、エッジ数を0からn-1まで増やし、各エッジ数パターンにおけるコストを比較して最小コストを更新していく（aka. edge relaxation）**ことです。エッジ数の各パターンに使われるコストというのはDPからみて問題を分解している部分に当たります。

![](https://storage.googleapis.com/zenn-user-upload/18e5d6217361-20230506.gif)
出典：https://commons.wikimedia.org/wiki/File:Bellman%E2%80%93Ford_algorithm_example.gif

マイナスウェイトのサイクルのケースはさておき、今はマイナスウェイトサイクルが存在しないグラフを想定してステップを説明します。
- 始点からすべての頂点に対して、最短経路のコストを、始点に対しては0、それ以外の頂点に対しては`+infinity`として初期化します
- グラフの頂点数-1回まで、グラフ内のすべてのエッジに対して以下の操作を繰り返します：
  - 各エッジ`(u, v)`に対して、頂点uからvへの最小コストを更新します。
  - つまり、`dist[v] > dist[u] + weight(u, v)` であれば、`dist[v] = dist[u] + weight(u, v)` と更新。

言葉だけでは理解しにくいので、実際の例を見た方が良いかもしれません。下記のグラフを考えてみてください。

![](https://storage.googleapis.com/zenn-user-upload/fe459399f934-20230506.png)

このグラフに対して、始点をノードAだと仮定し、Aから各ノードまでの最小コストを計算することを目的とします。サイクルはありますが、マイナスウェイトのサイクルが存在しません。

まずはDPテーブルを次のように初期化します。

| edge count | A   | B    | C    | D    |
| ---------- | --- | ---- | ---- | ---- |
| 0          | 0   | +inf | +inf | +inf |
| 1          | 0   | +inf | +inf | +inf |
| 2          | 0   | +inf | +inf | +inf |
| 3          | 0   | +inf | +inf | +inf |

ここで、`dp[0]["A"]`で取得した値というのは、始点から0本のエッジを使って、ノードAまでの合計最小コストを意味します。同様に、`dp[2]["C"]`というのは、始点から0-2本のエッジを使って、ノードCまでの合計最小コストとなります。

これでループを始めると、まずエッジ数が0の場合はどうなるかというと、0本を使うので、自分自身以外に到達することが不可能で、結局`dp[0]["A"]`以外は全部`+inf`となります。

エッジ数が1の場合はどうでしょう。使えるエッジは0-1本となりますが、1本を使う場合、
- 始点AからAまではコスト0のまま
- 始点AからBまではウェイト10のエッジが存在→0本の場合の`+inf`より小さいので、最小コストを10として更新
- 始点AからCまではウェイト50のエッジが存在→Bと同様に最小コストを50として更新
- 始点AからDまでの最小コストは上記と同様20として更新

これで一周が終わると、DPテーブルは以下と更新されました。

| edge count | A   | B    | C    | D    |
| ---------- | --- | ---- | ---- | ---- |
| 0          | 0   | +inf | +inf | +inf |
| 1          | 0   | 10   | 50   | 20   |
| 2          | 0   | +inf | +inf | +inf |
| 3          | 0   | +inf | +inf | +inf |

次にエッジ数が2の場合を見ます。使えるエッジは0-2本となりますが、すでに0と1本を使う場合の最小コストが得られているので、ここは2本を使う場合を考えるだけで良いのです。
- 始点AからAまでのコストは0のまま
- 始点AからBまでは、仮にエッジ2本を使うとA->D->Bの順で行ける→AからDまで1本を使うと20なので、DからBまでの-15と加算して5となる→現在最小コストは10なので、5として更新
- 始点AからCまでは、仮にエッジ2本を使うとA->B->Cの順で行ける→AからBまで1本を使うと10なので、BからCまでの10と加算して20となる→現在最小コストは50なので、20として更新
- 始点AからDまでは、仮にエッジ2本を使うとA->C->Dの順で行ける→AからCまで1本を使うと50なので、CからDまでの10と加算して60となる→現在最小コストは20なので、更新は不要

これで二週目が終わると、DPテーブルは以下と更新されました。

| edge count | A   | B    | C    | D    |
| ---------- | --- | ---- | ---- | ---- |
| 0          | 0   | +inf | +inf | +inf |
| 1          | 0   | 10   | 50   | 20   |
| 2          | 0   | 5    | 20   | 20   |
| 3          | 0   | +inf | +inf | +inf |

最後はエッジ数が3の場合を見ます。上記と同様に、エッジ2本の最小値はすでに前のステップで計算済みなので、ここは3本を使う場合のみを考えます。
- 始点AからAまでのコストは0のまま
- 始点AからBまでは、エッジ3本を使って行けるルートがないので、更新は不要
- 始点AからCまでは、仮にエッジ3本を使うとA->D->B->Cの順で行ける→AからBまで2本を使うと5なので、BからCまでの10と加算して15となる→現在最小コストは25なので、15として更新
- 始点AからDまでは、仮にエッジ3本を使うとA->B->C->Dの順で行ける→AからCまで2本を使うと20なので、CからDまでの10と加算して30となる→現在最小コストは20なので、更新は不要

最終的にDPテーブルは以下となります。

| edge count | A   | B    | C    | D    |
| ---------- | --- | ---- | ---- | ---- |
| 0          | 0   | +inf | +inf | +inf |
| 1          | 0   | 10   | 50   | 20   |
| 2          | 0   | 5    | 20   | 20   |
| 3          | 0   | 5    | 15   | 20   |

これで、始点AからBまでの最小コストは5、Cまでの最小コストは15、Dまでの最小コストは20というふうに結論付けられます。

### コード化

上記の例をコード化するとどうなるか、見てみましょう（マイナスウェイトのサイクルの検知についてまた後の節で）。

```js
function bellmanFord(graph, start) {
  const n = getSizeFromEdges(graph);
  const dp = Array.from({ length: n }, () => Array(n).fill(Infinity));

  // Initialize the DP table
  dp[0][start] = 0;

  // 0からn-1までループする
  for (let i = 1; i < n; i++) {
    // グラフはエッジリストで表現
    for (const [src, dst, weight] of graph) { 
    // 前の行の最小コストに、現在のウェイトを加算して判断する
      const prevMinCost = dp[i - 1][src]
      if (prevMinCost + weight < dp[i][dst]) {
        dp[i][dst] = prevMinCost + weight;
      }
    }
  }

  // TODO: Check for negative weight cycles

  // 最後の行のデータが始点から各ノードまでの最小コスト
  return dp[n - 1];
}

// エッジリストからノードの数を計算する
function getSizeFromEdges(edges) {
  // ノードの値はユニークとの前提
  const nodes = new Set();
  for (const [u, v] of graph) {
    nodes.add(u);
    nodes.add(v);
  }
  return nodes.size;
}
```

時間複雑度について、全てのノードがお互いに繋いでいる場合（最悪のケース）、`O(V*E)`となります。空間複雑度は、DPテーブルの`O(V*V)`となるのです。

### DPテーブルを最適化

上記の例ですでにわかるかもしれませんが、毎回のループにおいて、必要とされるのは前回のループにおける各最小コストだけとなっています。仮にノードは100個あるとすれば、上記のやり方では100行のデータを保存する必要がありますが、事実上前回分と今回分の2行のみで行けるはず、とのことです。つまり、上記の空間複雑度を、マトリックスの`O(V*V)`から配列の`O(V)`まで最適化することが可能です。

```js
function bellmanFord(graph, start) {
  const n = getSizeFromEdges(graph);
  const dist = Array(n).fill(Infinity);

  dist[start] = 0;

  for (let i = 1; i < n; i++) {
    const prevDist = [...dist]; // 前の行のデータとしてコピー
    for (const [src, dst, weight] of graph) {
      if (prevDist[src] + weight < dist[dst]) {
        dist[dst] = prevDist[src] + weight;
      }
    }
  }

  return dist;
}
```

時間複雑度にも最適化する余地はあります。仮にエッジ2本以降のイテレーションでは、3本でも4本でも更新が行わなかったら、2本の時点ですでに最小コストになっていることがわかります。すると、**更新が行われたかどうかを追跡する**ことで、ループから早期ブレークすることが可能です。

```js
function bellmanFord(graph, start) {
  const n = getSizeFromEdges(graph);
  const dist = Array(n).fill(Infinity);

  dist[start] = 0;

  for (let i = 1; i < n; i++) {
    let updated = false;
    const prevDist = [...dist]; // 前の行のデータとしてコピー

    for (const [src, dst, weight] of graph) {
      if (prevDist[src] + weight < dist[dst]) {
        dist[dst] = prevDist[src] + weight;
        updated = true;
      }
    }

    if (!updated) break; // 更新がなければすでに最小コストに達している
  }

  return dist;
}
```

これで時間複雑度は、`<=O(V*E)`となりました。

### マイナスウェイトサイクル（負の重み閉路）の検知

では、マイナスウェイトのサイクルの存在はどのように検知できるのでしょうか。

上記のイテレーションが終わると、マイナスウェイトのサイクルが存在しなければ、最後の行のデータは最小コストになっているはずです。逆に言えば、ループ終了後にいずれかのノードsrcからいずれかのノードdstまでの最小コスト`prev[src]`に、エッジで定義されたウェイトを乗せて、該当ノードまでの最小コスト`prev[dst]`よりも小さくなれば、無限に小さくなることができ、マイナスウェイトサイクルの存在がわかります。

```js
function bellmanFord(graph, start) {
  const n = getSizeFromEdges(graph);
  const dist = Array(n).fill(Infinity);

  dist[start] = 0;

  while (true) {
    let updated = false;
    const prevDist = [...dist]; // 前の行のデータとしてコピー

    for (const [src, dst, weight] of graph) {
      if (prevDist[src] + weight < dist[dst]) {
        dist[dst] = prevDist[src] + weight;
        updated = true;
      }
    }

    if (!updated) break; // 更新がなければすでに最小コストに達している
  }

  // 終了後イテレーションを追加
  for (const [src, dst, weight] of graph) {
    // すでに最小であればここはあり得ないはず
    if (prev[src] + weight < prev[dst]) {
      throw new Error("Graph contains a negative-weight cycle");
    }
  }

  return prev;
}
```

## SPFA Algorithm

ベルマンフォード法に対して、さらに最適化したSPFA(Shortest Path Faster Algorithm)アルゴリズムがあります。一言で言えば、DPテーブル・配列の代わりに、**キューを利用して各ノードまでの最小コストを更新するベルマンフォード法の改善版**です。

ステップは以下となります。
- 始点からすべての頂点に対して、最短経路のコストを、始点に対しては0、それ以外の頂点に対しては`+infinity`として初期化します
- 始点ノードをキューに追加します
- キューがからになるまで、以下の操作を繰り返します：
  - キューからノード`u`を取得します
  - ノード`u`に隣接する全ての頂点をループ
    - `u`から隣接の`v`までのウェイトを`dist[u]`へ加算し、現在の`dist[v]`と比較します
    - `dist[v] > dist[u] + weight(u, v)` であれば、`dist[v] = dist[u] + weight(u, v)` と更新。
    - 同時に更新されたノード`v`をキューにプッシュ

```js
function spfa(graph, start) {
  const n = getSizeFromEdges(graph);

  const dist = new Array(n).fill(Infinity);
  dist[start] = 0;

  // 必要なノードのみキューに追加する
  const queue = [start];

  while (queue.length > 0) {
    const u = queue.shift();

    for (const [src, dst, weight] of graph) {
      if (src === u) {
        const v = dst;

        if (dist[u] + weight < dist[v]) {
          dist[v] = dist[u] + weight;
          // 更新が行われたノードをキューに追加
          queue.push(v);
        }
      }
    }
  }

  // マイナスウェイトのサイクルチェック...

  return dist;
}
```

もちろん、グラフをエッジリストではなく、隣接リストの形で表現すると、よりすっきりに見えますし、隣接するノードだけループするので、スパースグラフの場合効率は良くなります。

```js
function spfa(graph, source) {
  const n = graph.length;
  const dist = Array(n).fill(Infinity);
  // すでにキューに存在するノードの重複プッシュを避けるために
  const inQueue = Array(n).fill(false);
  const queue = [];

  dist[source] = 0;
  queue.push(source);
  inQueue[source] = true;

  while (queue.length > 0) {
    const u = queue.shift();
    inQueue[u] = false;

    for (const [v, weight] of graph[u]) {
      const newDist = dist[u] + weight;
      if (newDist < dist[v]) {
        dist[v] = newDist;
        if (!inQueue[v]) {
          queue.push(v);
          inQueue[v] = true;
        }
      }
    }
  }

  return dist;
}
```

時間複雑度について、最悪のケースはベルマンフォード法と同じく`O(V*E)`となりますが、キューに入っているノードのみに対してループするので、通常は`O(E)`に近い効率になるらしいです（が、直接な証明と出典が見つかっていません）。空間について同じく`O(V)`となります。

## 実践問題

まずは[743. Network Delay Time](https://leetcode.com/problems/network-delay-time/description/)を見てみましょう。

![](https://storage.googleapis.com/zenn-user-upload/40ca98244edf-20230506.png)

基本マイナスウェイトがなければDijkstra法かBellman-Ford法のいずれか解決できますが、まずはDijkstraをみてみ。隣接リストの方でやりやすいのでまずは`times`から作成した方が良いでしょう。

```python
import heapq

def networkDelayTime(times, n, k):
    # グラフを表す隣接リストを作成
    graph = {}
    for u, v, w in times:
        if u not in graph:
            graph[u] = [j
        graph[u].append((v, w))

    pq = [(0, k)]  # heapでの保存形式は[cost, node]
    # 最短パスを保存するためのコスト配列をinfinityとして初期化
    dist = [float('inf')] * (n + 1)
    dist[k] = 0

    # ヒープにデータがある限り続ける
    while pq:
        dist, node = heapq.heappop(pq)
        # 既存の最小コストよりも大きい場合はスキップ
        if dist > dist[node]:
            continue
        # 現在のノードに隣接するノードがある場合はそれらを探索
        if node in graph:
            for neighbor, weight in graph[node]:
                newDist = dist + weight
                # 現存の最小コストより小さい場合は値を更新
                if newDist < dist[neighbor]:
                    dist[neighbor] = newDist
                    heapq.heappush(pq, (newDist, neighbor))

    # スタートノードからの最大距離を取得、0番はいらないので注意
    maxDist = max(dist[1:])

    # すべてのノードが到達可能＝infinityがない状態であれば最大コストをリターン
    return -1 if maxDist == float('inf') else maxDist
```

次に[787. Cheapest Flights Within K Stops](https://leetcode.com/problems/cheapest-flights-within-k-stops/description/)をみてみます。

![](https://storage.googleapis.com/zenn-user-upload/64b76eaa88d3-20230506.png)

この問題はDijkstra法とBellman-Ford法のいずれかで解決できますが、ここはBellman-Ford法で。

```js
function findCheapestPrice(n, flights, src, dst, k) {
  // 初期化
  // 最短距離を保存するための配列 (1次元)
  const dist = Array(n).fill(Infinity);
  dist[src] = 0; // スタートノードの距離は0

  // K回の中間ノードを使って探索
  for (let i = 0; i < k + 1; i++) {
    // 1つ前の距離を保存する配列を作成
    const prevDist = [...dist];

    // 全てのフライトに対して、更新可能な距離を確認
    for (const [u, v, w] of flights) {
      // 現在の距離よりも、新しい経路を使った距離が短い場合、距離を更新
      if (prevDist[u] + w < dist[v]) {
        dist[v] = prevDist[u] + w;
      }
    }
  }

  // もし目的地への距離が無限大でなければ、その距離を返す
  // そうでなければ、-1を返す
  return dist[dst] !== Infinity ? dist[dst] : -1;
}
```


## まとめ

これで単一始点最短経路問題についてよく見られる3つのアルゴリズムについて説明してきました。それぞれの時間複雑度と空間複雑度、およびグラフの特性（有向/無向、重み付き/非重み付き、負の重みの有無、負の重みのサイクルの有無）に基づいて適用可能なアルゴリズムをまとめました。

| アルゴリズム         | 時間複雑度                | 空間複雑度 | 有向 | 無向 | 重み付き | 非重み付き | 負の重み | 負の重みのサイクル |
| -------------------- | ------------------------- | ---------- | ---- | ---- | -------- | ---------- | -------- | ------------------ |
| ダイクストラ法       | `O(V^2)` or `O(E+V*logV)` | `O(V)`     | ◯    | ◯    | ◯        | ◯          | ×        | ×                  |
| ベルマン・フォード法 | `O(V*E)`                  | `O(V)`     | ◯    | ◯    | ◯        | ◯          | ◯        | △*                 |
| SPFA                 | `O(E)`~`O(VE)`            | `O(V)`     | ◯    | ◯    | ◯        | ◯          | ◯        | △*                 |

*ベルマン・フォード法およびSPFAは、マイナスウェイトのサイクルを検出できますが、マイナスウェイトのサイクルが存在する場合、最短経路が存在しないため、結果は求められません。

この辺りのアルゴリズムは、ヒープやDPの前知識が必要となっているので、比較的に難しい方かなと思います。自分も理解するために結構苦労しましたが、わかった後の気持ちは鬱憤を払ったような清々しさになっていて、価値はあると思いました（笑）。

次回は最終回の全点対最短経路問題について書いてみたいと思います。ではでは。

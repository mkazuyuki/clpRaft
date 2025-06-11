# AppendEntries 複製応答による nextIndex[]・matchIndex[] の誤更新とその対策

Leader は nextIndex[] matchIndex[] を管理せねばならない。

考えられるケース:
もし、AppendEntries RPC の返信が何らの原因（例えばネットワークの遅延やパケットの再送機構の誤動作）によって複製され、Leader がその複製された返信を処理することで nextIndex や matchIndex が誤って増加する可能性がある。これは、Follower が未適用のログエントリを受け入れたと誤認する状況を引き起こす可能性がある。

Leader のログに Follower のログが追い付いていない状況を判定し、管理する方法は以下の部分に現れている。

Rules for Servers > Leaders:
> - If last log index ≥ nextIndex for a follower: send AppendEntries RPC with log entries starting at nextIndex
>   • If successful: update nextIndex and matchIndex for follower(§5.3)
>   • If AppendEntries fails because of log inconsistency: decrement nextIndex and retry(§5.3)

これは個々の AppendEntries の成否が判ることを前提としている。だが、UDP で AE を行う場合、処理遅延やパケットの複製等が起こりうるから Leader の AE「送信」と Follower の「返信」が一対一対応になる保障はない。

- follower から返信が返る保障はない。
- 複製された AppendEntries、複製された返信 が返る可能性はゼロではない。

AEが複製されると、当該 follower の nextIndex, matchIndex を複数回更新してしまう。
1回の AE で index 一つ分のログを送信するなら、複製された回数に応じて nextIndex, matchIndex を increment | decrement してしまう(という実装が容易に想像される)。
# ここに Byzantine Fault Tollelant ではないことが現れているといえるか?

一回の AE 送信に 複製された複数回の AE 返信を想定する。
複製された返信により 誤って increment された nextIndex, matchIndex が declement されるケースがあるとすれば?

この場合↓ nextIndex は decrement され retry となる。
> • If AppendEntries fails because of log inconsistency:
>   decrement nextIndex and retry(§5.3)

AE に false 返信をする条件は以下の通り。つまり follower が Leader の持つログをまだ持っていない場合が典型的であろう。
AppendEntries RPC > Receiver implementation:
> 1. Reply false if term < currentTerm (§5.1)
> 2. Reply false if log doesn’t contain an entry at prevLogIndex whose term matches prevLogTerm (§5.3)

この条件では、誤って increment された nextIndex が AEに対する false 返信で decrement されることは期待できない。

---
Leader got a req from a client.

  1 2 3 
L 1 1 1
F 1 1

L nextIndex[F]=3 
  matchIndex[F]=2
---
AE(prevLogIndex=2, prevLogTerm=1)
---
  1 2 3 
L 1 1 1
F 1 1 1

AE dupulicates res success=True x2
Leader mistakenly update (increment) nextIndex, matchIndex.

L nextIndex[F] = 5
  matchIndex[F] = 4
---
Leader got a req from a client.

  1 2 3 4
L 1 1 1 1
F 1 1 1

---
AE(prevLogIndex=3, prevLogTerm=1)
---
  1 2 3 4
L 1 1 1 1
F 1 1 1 1

AE dupulicate res success=True x2
Leader mistakenly update (increment) nextIndex, matchIndex.

L nextIndex[F] = 7
  matchIndex[F] = 6

---

リーダーにおいて ある follower の nextIndex, matchIndex が log[] の index より大きな値にならないような対処、あるいは、なってしまった場合への 対処が必要。

一例として、AE の res に やり取りした ログの Index をつけ、res に載る Index で nextIndex, matchIndex を更新する方法があろう。
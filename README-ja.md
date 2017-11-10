# Ethereum Contract セキュリティ・テクニック＆Tips

このドキュメントは
[ConsenSys/smart-contract-best-practices](https://github.com/ConsenSys/smart-contract-best-practices)
を日本語訳したものです。  
**日本語の翻訳や訳文の改善は大歓迎です。お気軽にPRをください。**

[![Join the chat at https://gitter.im/ConsenSys/smart-contract-best-practices](https://badges.gitter.im/ConsenSys/smart-contract-best-practices.svg)](https://gitter.im/ConsenSys/smart-contract-best-practices?utm_source=badge&utm_medium=badge&utm_campaign=pr-badge&utm_content=badge)

目次:

<!-- TOC built using Sublime's MarkdownTOC plugin -->
<!-- MarkdownTOC -->

- [一般的な知識](#general-philosophy)
  - [根本的なトレードオフ: シンプル vs 複雑](#fundamental-tradeoffs-simplicity-versus-complexity-cases)
- [セキュリティに関する告知](#security-notifications) 
- [Solidityにおけるスマートコントラクトセキュリティための推奨事項](#recommendations-for-smart-contract-security-in-solidity)
  - [外部呼び出し](#external-calls)
  - [`assert()` で恒常性を担保する](#enforce-invariants-with-assert)
  - [`assert()` と `require()` プロパティを使用する](#use-assert-and-require-properly)
  - [integer の小数点以下切り捨てに注意](#beware-rounding-with-integer-division)
  - [Etherを強制的に送信できることに注意](#remember-that-ether-can-be-forcibly-sent-to-an-account)
  - [コントラクトが残高0Etherで作成されるとは限らない](#dont-assume-contracts-are-created-with-zero-balance)
  - [オンチェーンのデータはPublicであることに注意](#remember-that-on-chain-data-is-public)
  - [抽象コントラクトとインターフェースとのトレードオフに注意](#be-aware-of-the-tradeoffs-between-abstract-contracts-and-interfaces)
  - [複数のコントラクトを連携する場合、あるコントラクトが無反応で何もreturnしない可能性に注意](#in-2-party-or-n-party-contracts-beware-of-the-possibility-that-some-participants-may-drop-offline-and-not-return)
  - [fallback functionsはシンプルに保つ](#keep-fallback-functions-simple)
  - [関数と変数のスコープは明示的に宣言する](#explicitly-mark-visibility-in-functions-and-state-variables)
  - [pragmaを特定のコンパイラのバージョンにロックする](#lock-pragmas-to-specific-compiler-version)
  - [ゼロ除算に注意 \(Solidity < 0.4\)](#beware-division-by-zero-solidity--04)
  - [関数とイベントを区別する](#differentiate-functions-and-events)
  - [より新しいSolidity構文を好む](#prefer-newer-solidity-constructs)
- [Known Attacks](#known-attacks)
  - [Race Conditions\*](#race-conditions%5C)
  - [Transaction-Ordering Dependence \(TOD\) / Front Running](#transaction-ordering-dependence-tod--front-running)
  - [Timestamp Dependence](#timestamp-dependence)
  - [Integer Overflow and Underflow](#integer-overflow-and-underflow)
  - [DoS with \(Unexpected\) revert](#dos-with-unexpected-revert)
  - [DoS with Block Gas Limit](#dos-with-block-gas-limit)
  - [~~Call Depth Attack~~](#%7E%7Ecall-depth-attack%7E%7E)
- [Software Engineering Techniques](#software-engineering-techniques)
  - [Upgrading Broken Contracts](#upgrading-broken-contracts)
  - [Circuit Breakers \(Pause contract functionality\)](#circuit-breakers-pause-contract-functionality)
  - [Speed Bumps \(Delay contract actions\)](#speed-bumps-delay-contract-actions)
  - [Rate Limiting](#rate-limiting)
  - [Contract Rollout](#contract-rollout)
  - [Bug Bounty Programs](#bug-bounty-programs)
- [Security-related Documentation and Procedures](#security-related-documentation-and-procedures)
- [Security Tools](#security-tools)
  - [Linters](#linters)
- [Future improvements](#future-improvements)
- [Smart Contract Security Bibliography](#smart-contract-security-bibliography)
- [Reviewers](#reviewers)
- [License](#license)

<!-- /MarkdownTOC -->

このドキュメントは中堅Solidityプログラマにセキュリティの基礎を伝えるために作られています。  
プルリクエストは大歓迎です。
小さな修正からセクションの追加、あるいは記事やブログを書いた場合は
[参考ドキュメント](#bibliography)に追加してください。  
詳しくは[Contribution Guidelines](CONTRIBUTING.md)を確認してください。  
(注: 内容の追加などは
[翻訳元](https://github.com/ConsenSys/smart-contract-best-practices)
へプルリクエストを投げてください)

#### 特に歓迎する項目

以下の領域のコンテンツは特に歓迎します:

- Solidityコードのテスト
- スマートコントラクトやブロックチェーンベースのプログラミングの、ソフトウェア開発チュートリアル

<a name="general-philosophy"></a>

## 一般的な知識

Ethereumや複雑なブロックチェーンのプログラムは歴史が浅く、極めて実験的な段階です。  
したがって新しいバグやセキュリティ上のリスクが発見されたり、
新しいベストプラクティスが開発されるなど、セキュリティの領域は常に変化してゆくと認識する必要があります。    

スマートコントラクトのプログラミングには、あなたの今までの経験とは全く異なるマインドセットが必要です。  
1つのミスが多大な損害を与え、かつその修正は困難な場合があります。
これはWEB開発よりも、むしろ組み込み開発や金融系の開発によく似ています。
既知の脆弱性に備えるだけでは不充分であり、
新しい開発手法を身につける必要があります。

- **危機に備える**. ある程度の規模のContractであれば何らかの不具合を含みます。そのため、バグや脆弱性が発見された際にしっかりと対応できるコードを書く必要があります。
  - 危機的な状況が発生したらContractを一時停止する("サーキットブレーカー")
  - 危機に備えて資金量を管理する (レート制限, 上限量)
  - バグFIXや改善のためにアップグレード方法を準備する

- [**慎重に運用開始する**.](#contract-rollout) バグを本番環境リリース前に潰しておくことは最善の打ち手です。
  - Contractを徹底的にテストし、新しい攻撃手法へのテストを常に追加してゆく。
  - [バグ報奨金制度](#bounties) をテストネットでのアルファリリースから提供する。
  - 段階的に運用開始し、すべてのフェーズでテストとユーザを増やす

- **Contractをシンプルに保つ**. 複雑さは不具合の可能性を高めます。
  - Contractのロジックがシンプルであることを担保する
  - Contractと関数を小さく保つためにモジュラー化する
  - 可能な限り既存のツールやコードを使用する (例. ランダム数を取得する処理を書かない)
  - 処理は可能な限りわかりやすく書く
  - ブロックチェーンは、システムで非中央集権性が求められる部分にだけ適用する

- **常に最新の情報を追う**. 新しいセキュリティ情報を追うために、次のセクションにまとめた一覧を活用してください。
  - あなたのcontractに新しく発見されたバグが含まれないか確認する
  - 可及的速やかに全てのツールやライブラリを最新バージョンへ更新する
  - 有用と思われる新しいセキュリティ対策を採用する

- **ブロックチェーンの特性に注意する**. あなたのプログラミング経験の多くはEthereumの開発にも通用しますが、いくつか注意すべき罠があります。
  - 外部のcontractをコールする場合は特に注意する。悪意あるコードが実行され制御フローが変更される可能性がある。
  - Publicな関数がPublicであることをきちんと理解する。悪意を持って実行される可能性がある。またPrivateなデータであっても、誰からも参照されうることを認識する。
  - gasのコストと、ブロックのgas limitに注意する。

### 根本的なトレードオフ: シンプル vs 複雑
<a name="fundamental-tradeoffs-simplicity-versus-complexity-cases"></a>

スマートコントラクトの構造とセキュリティを考える場合、そこには複数の根本的なトレードオフがあります。  
いかなるスマートコントラクト・システムであってもこれらのトレードオフを適切なバランスとすることを推奨します。

しかし、そこにはセキュリティとソフトウェア・エンジニアリング・ベストプラクティスの両方を担保できない重用な例外があります。  
これらのケースでは、以下のようなスマートコントラクトの特性を事前に認識しておくことで最良の組み合わせを選択できます。

- アップグレード不可 vs アップグレード可能
- モノリシック vs モジュラー
- 重複 vs 再利用

#### アップグレード不可 vs アップグレード可能

多くのドキュメントで、Killable、アップグレード可能、変更可能などのパターンが強調されます。しかしそこにはセキュリティと柔軟性の根本的なトレードオフが存在します。

柔軟性の高い設計は複雑性を増し、攻撃の可能性を高めます。  
シンプルさは複雑さよりも、極めて限定的な機能を限られた期間提供するスマートコントラクトシステムで特に効果的です。例としてはトークンセール(ICO)システムや、governance-free、finite-time-frame、などがあげられます。

#### モノリシック vs モジュラー


モノリシックに全てを詰め込んだコントラクトは、処理に必要なすべての情報を内包し把握しておけます。モノリシックに作成されたスマートコントラクトシステムで高い評価を受けることは稀ですが、たとえばコードレビューの効率化・最適化など、データと処理の流れを極端にローカルに留める手法には議論の余地があります。  

この点も、他のトレードオフと同様に検討しましょう。セキュリティ・ベストプラクティスは期間限定のコントラクトに適した方法と、より複雑で利用期間の長いコントラクトシステムでは異なります。


#### 重複 vs 再利用

ソフトウェア・エンジニアリングの観点からは、スマートコントラクトシステムは可能な限り再利用可能であることが望ましいと言えます。Solidityではコントラクトコードを再利用するための複数の方法があります。 **自分でデプロイした** 既存のコントラクトは、一般的にコントラクトを再利用する上で最も安全な方法です。

**重複** は、自分でデプロイしたコントラクトが利用できない場合にしばしば有効です。[Live Libs](https://github.com/ConsenSys/live-libs) や [Zeppelin Solidity](https://github.com/OpenZeppelin/zeppelin-solidity)は、セキュアなコード・パターンの提供を模索しています。コントラクトのセキュリティ分析では、再利用コードであっても対象のスマートコントラクトシステムで扱う資金に釣り合ったレベルの信頼性が求められます。

<a name="security-notifications"></a>

## セキュリティに関する告知

これは、しばしばEthereumやSolidityのセキュリティに関する情報が掲載されるリソースのリストです。公式なセキュリティに関する告知はオフィシャルブログで発表されますが、多くの場合、脆弱性はその他の場所でいち早く公開・議論されます。

- [Ethereum Blog](https://blog.ethereum.org/): Ethereumのオフィシャルブログ
  - [Ethereum Blog - Security only](https://blog.ethereum.org/category/security/): **Security** タグの付いたブログポスト
- [Ethereum Gitter](https://gitter.im/orgs/ethereum/rooms) チャットルーム
  - [Solidity](https://gitter.im/ethereum/solidity)
  - [Go-Ethereum](https://gitter.im/ethereum/go-ethereum)
  - [CPP-Ethereum](https://gitter.im/ethereum/cpp-ethereum)
  - [Research](https://gitter.im/ethereum/research)
- [Reddit](https://www.reddit.com/r/ethereum)
- [Network Stats](https://ethstats.net/)

あなたのコントラクトに関連する情報が掲載される可能性があるため、これら全てのリソースを **定期的に** チェックすることを強く推奨します。
  
また、以下のリストはセキュリティについて言及する可能性のあるEthereumのコア・デベロッパーたちです。[bibliography](https://github.com/ConsenSys/smart-contract-best-practices#smart-contract-security-bibliography) にはより多くのコミュニティからの情報がまとめられています。

- **Vitalik Buterin**: [Twitter](https://twitter.com/vitalikbuterin), [Github](https://github.com/vbuterin), [Reddit](https://www.reddit.com/user/vbuterin), [Ethereum Blog](https://blog.ethereum.org/author/vitalik-buterin/)
- **Dr. Christian Reitwiessner**: [Twitter](https://twitter.com/ethchris), [Github](https://github.com/chriseth), [Ethereum Blog](https://blog.ethereum.org/author/christian_r/)
- **Dr. Gavin Wood**: [Twitter](https://twitter.com/gavofyork), [Blog](http://gavwood.com/), [Github](https://github.com/gavofyork)
- **Vlad Zamfir**: [Twitter](https://twitter.com/vladzamfir), [Github](https://github.com/vladzamfir), [Ethereum Blog](https://blog.ethereum.org/author/vlad/)

コア・デベロッパーを通じて、ブロックチェーンに関連した広く重用なセキュリティ周りのコミュニティにリーチできるでしょう。セキュリティについての情報公開や観測状況が各所から寄せられます。


<a name="recommendations-for-smart-contract-security-in-solidity"></a>

## Solidityにおけるスマートコントラクトセキュリティための推奨事項

<a name="external-calls"></a>

### 外部呼び出し

#### 可能であれば外部呼び出しは避ける
<a name="avoid-external-calls"></a>

信頼されていないコントラクトを呼び出すと、いくつかの予期しないリスクやエラーが発生する可能性があります。
外部呼び出しはコントラクトや依存する他のコントラクトで悪意のあるコードを実行する可能性があります。
したがって、すべての外部コールは潜在的なセキュリティリスクとして扱われ、可能であれば削除されるべきです。 外部コールを削除できない場合は、このセクションの残りのセクションの推奨事項を使用して危険を最小限に抑えてください。

<a name="send-vs-call-value"></a>

#### `send()`、`transfer()`、`call.value()()`の間のトレードオフに注意

Etherを送信する時は `someAddress.send()`、 `someAddress.transfer()`、 `someAddress.call.value()()` の使用において、相対的なトレードオフに注意して下さい。

- `x.transfer(y)` は `require(x.send(y));` と等価です。sendはtransferとは対称的に低レベルのものなので、可能な限りtransferを使うことをおすすめします。
- `someAddress.send()` と `someAddress.transfer()` は[reentrancy](#reentrancy)に対して安全と考えられています。これらのメソッドがコード実行を引き起こしている間、呼び出されるコントラクトには、現在イベントを記録するのに十分な2,300gasの報酬のみが与えられます。
- `someAddress.call.value（）（）`は、提供されたetherとトリガーコードの実行を送信します。実行されたコードには実行可能なすべてのガスが与えられており、このタイプの値の転送はreentrancyに対して安全ではありません。

`send()` や `transfer()` を使うとreentrancyを防ぐことができますが、フォールバック関数が2,300を超えるガスを必要とするいくつかのコントラクトと合わないコストを支払って行います。

このトレードオフのバランスを取る1つのパターンは、[*push* と *pull*](#favor-pull-over-push-payments)メカニズムの両方を実装することです。pushコンポーネントに対して `send（）` または `transfer（）` を使用し、pullコンポーネントに対して `call.value（）（）` を使用します。

値の転送に `send（）`や `transfer（）` を排他的に使用することは、reentrancyに対して安全なコントラクトを作るのではなく、それらの特定の値の転送をreentrancyに対して安全にするだけです。

<a name="handle-external-errors"></a>

#### 外部呼び出しのエラーハンドリング

Solidityは `address.call()`、`address.callcode()`、`address.delegatecall()` および `address.send` のようなローアドレスで動作する低レベルの呼び出しメソッドを提供します。
これらの低レベルメソッドは決して例外をスローしませんが、呼び出しが例外を検出した場合は `false` を返します。
一方で `ExternalContract.doSomething（）` などのコントラクト呼び出しは自動的にスローを広める(例えば `doSomething（）` がスローした場合には `ExternalContract.doSomething（）` も同様に例外をスローします。低レベルのコールメソッドを使用する場合は、戻り値をチェックすることによって、呼び出しが失敗する可能性を確実に処理してください。

```
// bad
someAddress.send(55);
someAddress.call.value(55)(); // すべての残りのガスを送り、結果をチェックしないため二重に危険です
someAddress.call.value(100)(bytes4(sha3("deposit()"))); // depositが例外をスローすると、raw call() はfalseを返すだけでトランザクションはもとに戻されません

// good
if(!someAddress.send(55)) {
    // いくつかの失敗コード
}

ExternalContract(someAddress).deposit.value(100);
```

<a name="expect-control-flow-loss"></a>

#### 外部呼び出し後に制御フローを仮定するな

未処理の呼び出しやコントラクトの呼び出しを使用する場合は、`ExternalContract`が信頼できない場合に悪質なコードが実行されることを想定します。
`ExternalContract`が悪意のあるものではないとしても、呼び出すコントラクトによって悪質なコードが実行される可能性があります。
特に危険なのは、悪質なコードが制御フローを乗っ取ってレースコンディションに陥ることです。
(この問題の詳細な議論については[Race Conditions](https://github.com/ConsenSys/smart-contract-best-practices/#race-conditions)を参照して下さい)。

<a name="favor-pull-over-push-payments"></a>

#### 外部呼び出しのためのプッシュプルオーバープッシュ

外部呼び出しが誤って、または意図的に失敗する可能性があります。
このような障害によって引き起こされる被害を最小限に抑えるには、各外部呼び出しを呼び出しの受信者が開始できる独自のトランザクションに分離する方がよい場合があります。
これは、支払いに特に関連します。ユーザーが自動的に資金を押し出すのではなく、資金を引き出すことをお勧めします。
(これにより[gas limit問題](https://github.com/ConsenSys/smart-contract-best-practices/#dos-with-block-gas-limit)の可能性が減ります。)単一のトランザクションで複数の `send（）` 呼び出しを組み合わせることは避けてください。

```
// bad
contract auction {
    address highestBidder;
    uint highestBid;

    function bid() payable {
        require(msg.value >= highestBid);

        if (highestBidder != 0) {
            require(highestBidder.send(highestBid)) // この呼び出しが絶えず失敗した場合、誰も入札できません
        }

       highestBidder = msg.sender;
       highestBid = msg.value;
    }
}

// good
contract auction {
    address highestBidder;
    uint highestBid;
    mapping(address => uint) refunds;

    function bid() payable external {
        require(msg.value >= highestBid);

        if (highestBidder != 0) {
            refunds[highestBidder] += highestBid; // このユーザーが請求できる払い戻しを記録します
        }

        highestBidder = msg.sender;
        highestBid = msg.value;
    }

    function withdrawRefund() external {
        uint refund = refunds[msg.sender];
        refunds[msg.sender] = 0;
        require(msg.sender.send(refund)); // 送信失敗の場合は状態を元に戻す
        }
    }
}
```

<a name="mark-untrusted-contracts"></a>

#### 信用できないコントラクトをマークする

外部コントラクトと対話するときは、変数、メソッド、およびコントラクト・インターフェースに、それらとのインタラクションが潜在的に危険なものであることを明確にするような名前を付けます。

```
// bad
Bank.withdraw(100); // 信用できるかどうかは不明

function makeWithdrawal(uint amount) { // この機能が潜在的に危険であることが明らかでない
    Bank.withdraw(amount);
}

// good
UntrustedBank.withdraw(100); // 信用できない外部呼び出し
TrustedBank.withdraw(100); // XYZ社によってメンテナンスされた外部の信用銀行コントラクト

function makeUntrustedWithdrawal(uint amount) {
    UntrustedBank.withdraw(amount);
}
```

<a name="enforce-invariants-with-assert"></a>

### `assert()` で恒常性を担保する

アサーション(表明)は、変更されないはずのプロパティが改ざんされた場合など、表明違反(assertion failure)が発生した場合にシステムを保護するトリガーとして機能します。  
一例としては、トークンとetherの供給比率検証やトークン供給コントラクト等に有効です。
`assert()` を使用することで、対象の値が常に想定したものであることを検証できます。  
アサーションは、場合に応じてポージングやアップデート許可等のテクニックと併用すべきです。  
（そうしなければ、アサーションが常に失敗してしまう局面でも身動きが取れなくなってしまう可能性があります）

実装例:

```
contract Token {
    mapping(address => uint) public balanceOf;
    uint public totalSupply;

    function deposit() public payable {
        balanceOf[msg.sender] += msg.value;
        totalSupply += msg.value;
        assert(this.balance >= totalSupply);
    }
}
```

注意: このアサーションは厳密な検証では **ありません** 。コントラクトは `deposit()` ファンクションを介さずとも[強制的にetherを送信する](#ether-forcibly-sent)ことができるからです!

<a name="use-assert-and-require-properly"></a>

### `assert()` と `require()` プロパティを使用する

Solidity 0.4.10で `assert()` と `require()` が導入されました。  
`require(condition)` は、もし `condifiton` がfalseであればrevertされます。すべてのユーザーの入力値にはこのバリデーションを利用すべきです。  
`assert(condition)` も同様に、 `condition` がfalseである場合はrevertされます。しかし、こちらは内部エラーやコントラクトの異常を検知するために、不変であるべき値に対して使用します。  
公式検証ツールがリーチできない `invalid opcode` を検証できるようにするため、これらの仕組みを使用してください。

<a name="beware-rounding-with-integer-division"></a>

### integerの小数点以下切り捨てに注意

integerの小数点以下の値は切り捨てられます。より精度を求めるのであれば、掛け算(multiplier)を用いるか、分子(numerator)と分母(denominator)を保持してください。  
（将来的にSolidityは、この種の計算をより容易に行えるように固定点:fixed-pointをサポートする予定です）

```
// bad
uint x = 5 / 2; // 結果は2。 integerの小数点以下は常に切り捨てられます。

// good
uint multiplier = 10;
uint x = (5 * multiplier) / 2;

uint numerator = 5;
uint denominator = 2;
```

<a name="remember-that-ether-can-be-forcibly-sent-to-an-account"></a>

### Etherを強制的に送信できることに注意

コントラクトのEther残高(balance)を厳密にチェックするコードを書く場合は要注意です。

攻撃者はEther(wei)を強制的に任意のアカウントに送信でき、しかもこれは防止することができません。(フォールバック関数 `revert()` であっても)

攻撃者はコントラクトを作成し、1 weiだけそのコントラクトに保持させたのちに `selfdestruct(victimAddress)` を実行することでこの攻撃を実現できます。
この場合、攻撃対象のアドレス( `victimAddress` )ではいかなるコードも実行されません。そのためこの攻撃は防止不可能です。

<a name="dont-assume-contracts-are-created-with-zero-balance"></a>

### コントラクトが残高0Etherで作成されるとは限らない

攻撃者はEther(wei)を、コントラクトが作成される前に送信しておくことが可能です。

コントラクトは初期のEther残高がゼロであると思い込むべきではありません。詳細は [issue 61](https://github.com/ConsenSys/smart-contract-best-practices/issues/61) 
を参照してください。

<a name="remember-that-on-chain-data-is-public"></a>

### オンチェーンのデータはPublicであることに注意

多くのアプリケーションで、送信されたデータを特定の時点まで隠蔽しておく必要があります。よくある例としては、ゲーム(例: じゃんけん)や、オークション(例: 封印入札方式、セカンドプライス・オークション)が挙げられます。
もし情報の隠蔽が必要となるアプリケーションを作成するのであれば、ユーザが不適切なタイミングで情報を公開しないよう実装する必要があります。


例:

* じゃんけんの場合、最初にまずお互いの出す手(グー、チョキ、パー)をハッシュ化したデータを送信し、次にお互いの実際の手を送信します。もし送信された実際の手にハッシュとの齟齬が生じる場合、無効とします。
* オークションの場合、最初の段階でユーザーの指値価格をハッシュ化したデータを送信させます(それより前に指値価格よりも大きいデポジットが必要です)。次のフェーズで、実際の指値価格を送信させます。
* 乱数生成器に依存するアプリケーションを作成する場合、その処理の流れは次のようであるべきです。 (1) ユーザの手を送信 (2) 乱数生成 (3) ユーザの支払い。
  乱数生成の方法については活発な研究が行われています。現状で最良の解決策は、ビットコイン・ブロック・ヘッダ(http://btcrelay.org で検証済)、ハッシュ・コミット・公開スキーム(つまり、当事者の片方が乱数を生成し、そのハッシュを公開し、後で数値を明らかにする方法)、または[RANDAO](http://github.com/randao/randao)が挙げられます。
* 高頻度バッチオークション(frequent batch auction)を実装する場合も、やはりハッシュ・コミットスキームが求められます。

<a name="be-aware-of-the-tradeoffs-between-abstract-contracts-and-interfaces"></a>

### 抽象コントラクトとインターフェースとのトレードオフに注意

抽象コントラクトとインターフェイスは、ともにスマートコントラクトの再利用性とカスタマイズ性を高めるためのアプローチです。

インターフェイスはSolidity 0.4.11で実装された、抽象コントラクトに類似した概念ですが、関数を実装することができません。他にも、ストレージにアクセスできない、一般的に抽象コントラクトをより実用的にするために利用する、等の制限があります。しかしインターフェイスは、実際にコントラクトを実装する前の設計段階において有用です。

またコントラクトを抽象コントラクトとするのではない限り、抽象コントラクトを実装したコントラクトは必ず未実装の関数をオーバーライドによって実装しなければならず、それを強制するためにも役立ちます。

<a name="in-2-party-or-n-party-contracts-beware-of-the-possibility-that-some-participants-may-drop-offline-and-not-return"></a>

### 複数のコントラクトを連携する場合、あるコントラクトが無反応で何もreturnしない可能性に注意

払い戻しや引き出しの実装を、特殊な処理でしか資金を引き出せないコントラクトに依存させないでください。

たとえばじゃんけんゲームの場合、よくあるミスとしては両方のプレイヤーが手を出すまでは資金を引き出せない仕様とすることです。この場合、悪意のあるプレイヤーは、決して自分の手を出さないというシンプルな方法で相手の資金に打撃を与えることが可能です。実際、もし相手が先に手を出したことが確認できれば、その相手に損をさせようと意図する悪意あるプレイヤーは決して自分の手を開示しないでしょう。

この問題は、状況チャネル決済のコンテキストでも発生しうるものです。それが問題となる場合の対応策としては、たとえば（1）期限を設けてその期限までに参加しなかった参加者は無効とする、または（2）参加者が参加しているすべての状況で情報を提出するよう期待されている場合、金銭的なインセンティブを加えることを検討する、などが考えられます。

<a name="keep-fallback-functions-simple"></a>

### fallback functionsはシンプルに保つ

[Fallback functions](http://solidity.readthedocs.io/en/latest/contracts.html#fallback-function)は引数無しでメッセージを送られた場合(あるいはfunctionが存在しない場合)にコールされ、`.send()`や`.transfer()`からコールされた場合は2,300程度のガスしか使用できません。

もし`.send()`や`.transfer()`からEtherを受け取りたい場合、fallback functionで実装可能な処理はイベントのログを取る程度です。より多くのコンピューティングやガスが必要な処理の場合、別の関数を定義してそれを呼び出してください。

```
// bad
function() payable { balances[msg.sender] += msg.value; }

// good
function deposit() payable external { balances[msg.sender] += msg.value; }

function() payable { LogDepositReceived(msg.sender); }
```

<a name="explicitly-mark-visibility-in-functions-and-state-variables"></a>

### 関数と変数のスコープは明示的に宣言する

関数と変数のスコープは明示的に宣言しましょう。関数は宣言時に `external`, `public`, `internal`, または `private` を指定できます。これらの違いを理解してください。

たとえば、多くの場面では `public` ではなく `external` で充分でしょう。変数には `external` は使用できません。スコープを明示的にすることで、不適切なユーザが変数にアクセスしたり関数をコールできてしまうという問題をより容易に把握できます。

```
// bad
uint x; // デフォルトは private ですが、それでも明示すべきです
function buy() { // デフォルトは public となります
    // public code
}

// good
uint private y;
function buy() external {
    // 外部からしかコールできません
}

function utility() public {
    // 外部からも内部からもコールできます。このコードを変更する場合、そのどちらの側面も考慮する必要があります
}

function internalAction() internal {
    // 内部からしかコールできません
}
```

<a name="lock-pragmas"></a>

### pragmaを特定のコンパイラのバージョンにロックする

コントラクトは、最もよくテストされたものと同じコンパイラ・バージョンとフラグでデプロイする必要があります。
pragmaをロックすると、未知のバグのリスクがより高い最新のコンパイラなどを使用して、コントラクトが誤って展開されないようになります。
コントラクトは他の人によっても展開される可能性があり、pragmaは元の著者が意図したコンパイラのバージョンを示します。

```
// bad
pragma solidity ^0.4.4;


// good
pragma solidity 0.4.4;
```

<a name="beware-division-by-zero"></a>

### ゼロ除算に注意 (Solidity < 0.4)

0.4より前のバージョンでは、数値をゼロで割ったときにSolidityは[ゼロを返し](https://github.com/ethereum/solidity/issues/670)、例外をスローしません。
バージョン0.4以上で動作していることを確認してください。

<a name="differentiate-functions-events"></a>

### 関数とイベントを区別する

関数とイベントの混乱の危険を避けるため、先頭を大文字にしてイベントの前に接頭辞を付ける（*Log* を提案する）。
関数の場合は、コンストラクタを除き、常に小文字で始まります。

```
// bad
event Transfer() {}
function transfer() {}

// good
event LogTransfer() {}
function transfer() external {}
```

<a name="prefer-newer-constructs"></a>

### より新しいSolidity構文を好む

`selfdestruct`（`suicide`より新しい）と `keccak256`（`sha3`より新しい）のような構文/エイリアスを好みます。
`require（msg.sender.send（1 ether））`のようなパターンは `msg.sender.transfer（1 ether）`のように `transfer（）`を使って単純化することもできます。

<a name="known-attacks"></a>

## 既知の攻撃手法

<a name="race-conditions"></a>

### Race Conditions<sup><a href='#footnote-race-condition-terminology'>\*</a></sup>

One of the major dangers of calling external contracts is that they can take over the control flow, and make changes to your data that the calling function wasn't expecting. This class of bug can take many forms, and both of the major bugs that led to the DAO's collapse were bugs of this sort.

<a name="reentrancy"></a>

#### Reentrancy

The first version of this bug to be noticed involved functions that could be called repeatedly, before the first invocation of the function was finished. This may cause the different invocations of the function to interact in destructive ways.

```
// INSECURE
mapping (address => uint) private userBalances;

function withdrawBalance() public {
    uint amountToWithdraw = userBalances[msg.sender];
    require(msg.sender.call.value(amountToWithdraw)()); // At this point, the caller's code is executed, and can call withdrawBalance again
    userBalances[msg.sender] = 0;
}
```

Since the user's balance is not set to 0 until the very end of the function, the second (and later) invocations will still succeed, and will withdraw the balance over and over again. A very similar bug was one of the vulnerabilities in the DAO attack.

In the example given, the best way to avoid the problem is to [use `send()` instead of `call.value()()`](https://github.com/ConsenSys/smart-contract-best-practices#send-vs-call-value). This will prevent any external code from being executed.

However, if you can't remove the external call, the next simplest way to prevent this attack is to make sure you don't call an external function until you've done all the internal work you need to do:

```
mapping (address => uint) private userBalances;

function withdrawBalance() public {
    uint amountToWithdraw = userBalances[msg.sender];
    userBalances[msg.sender] = 0;
    require(msg.sender.call.value(amountToWithdraw)()); // The user's balance is already 0, so future invocations won't withdraw anything
}
```

Note that if you had another function which called `withdrawBalance()`, it would be potentially subject to the same attack, so you must treat any function which calls an untrusted contract as itself untrusted. See below for further discussion of potential solutions.

#### Cross-function Race Conditions

An attacker may also be able to do a similar attack using two different functions that share the same state.

```
// INSECURE
mapping (address => uint) private userBalances;

function transfer(address to, uint amount) {
    if (userBalances[msg.sender] >= amount) {
       userBalances[to] += amount;
       userBalances[msg.sender] -= amount;
    }
}

function withdrawBalance() public {
    uint amountToWithdraw = userBalances[msg.sender];
    require(msg.sender.call.value(amountToWithdraw)()); // At this point, the caller's code is executed, and can call transfer()
    userBalances[msg.sender] = 0;
}
```

In this case, the attacker calls `transfer()` when their code is executed on the external call in `withdrawBalance`. Since their balance has not yet been set to 0, they are able to transfer the tokens even though they already received the withdrawal. This vulnerability was also used in the DAO attack.

The same solutions will work, with the same caveats. Also note that in this example, both functions were part of the same contract. However, the same bug can occur across multiple contracts, if those contracts share state.

#### Pitfalls in Race Condition Solutions

Since race conditions can occur across multiple functions, and even multiple contracts, any solution aimed at preventing reentry will not be sufficient.

Instead, we have recommended finishing all internal work first, and only then calling the external function. This rule, if followed carefully, will allow you to avoid race conditions. However, you need to not only avoid calling external functions too soon, but also avoid calling functions which call external functions. For example, the following is insecure:

```
// INSECURE
mapping (address => uint) private userBalances;
mapping (address => bool) private claimedBonus;
mapping (address => uint) private rewardsForA;

function withdraw(address recipient) public {
    uint amountToWithdraw = userBalances[recipient];
    rewardsForA[recipient] = 0;
    require(recipient.call.value(amountToWithdraw)());
}

function getFirstWithdrawalBonus(address recipient) public {
    require(!claimedBonus[recipient]); // Each recipient should only be able to claim the bonus once

    rewardsForA[recipient] += 100;
    withdraw(recipient); // At this point, the caller will be able to execute getFirstWithdrawalBonus again.
    claimedBonus[recipient] = true;
}
```

Even though `getFirstWithdrawalBonus()` doesn't directly call an external contract, the call in `withdraw()` is enough to make it vulnerable to a race condition. you therefore need to treat `withdraw()` as if it were also untrusted.

```
mapping (address => uint) private userBalances;
mapping (address => bool) private claimedBonus;
mapping (address => uint) private rewardsForA;

function untrustedWithdraw(address recipient) public {
    uint amountToWithdraw = userBalances[recipient];
    rewardsForA[recipient] = 0;
    require(recipient.call.value(amountToWithdraw)());
}

function untrustedGetFirstWithdrawalBonus(address recipient) public {
    require(claimedBonus[recipient]); // Each recipient should only be able to claim the bonus once

    claimedBonus[recipient] = true;
    rewardsForA[recipient] += 100;
    untrustedWithdraw(recipient); // claimedBonus has been set to true, so reentry is impossible
}
```

In addition to the fix making reentry impossible, [untrusted functions have been marked.](https://github.com/ConsenSys/smart-contract-best-practices#mark-untrusted-contracts) This same pattern repeats at every level: since `untrustedGetFirstWithdrawalBonus()` calls `untrustedWithdraw()`, which calls an external contract, you must also treat `untrustedGetFirstWithdrawalBonus()` as insecure.

Another solution often suggested is a [mutex](https://en.wikipedia.org/wiki/Mutual_exclusion). This allows you to "lock" some state so it can only be changed by the owner of the lock. A simple example might look like this:

```
// Note: This is a rudimentary example, and mutexes are particularly useful where there is substantial logic and/or shared state
mapping (address => uint) private balances;
bool private lockBalances;

function deposit() payable public returns (bool) {
    if (!lockBalances) {
        lockBalances = true;
        balances[msg.sender] += msg.value;
        lockBalances = false;
        return true;
    }
    revert();
}

function withdraw(uint amount) payable public returns (bool) {
    if (!lockBalances && amount > 0 && balances[msg.sender] >= amount) {
        lockBalances = true;

        if (msg.sender.call(amount)()) { // Normally insecure, but the mutex saves it
          balances[msg.sender] -= amount;
        }

        lockBalances = false;
        return true;
    }

    revert();
}
```

If the user tries to call `withdraw()` again before the first call finishes, the lock will prevent it from having any effect. This can be an effective pattern, but it gets tricky when you have multiple contracts that need to cooperate. The following is insecure:

```
// INSECURE
contract StateHolder {
    uint private n;
    address private lockHolder;

    function getLock() {
        require(lockHolder == 0);
        lockHolder = msg.sender;
    }

    function releaseLock() {
        lockHolder = 0;
    }

    function set(uint newState) {
        require(msg.sender == lockHolder);
        n = newState;
    }
}
```

An attacker can call `getLock()`, and then never call `releaseLock()`. If they do this, then the contract will be locked forever, and no further changes will be able to be made. If you use mutexes to protect against race conditions, you will need to carefully ensure that there are no ways for a lock to be claimed and never released. (There are other potential dangers when programming with mutexes, such as deadlocks and livelocks. You should consult the large amount of literature already written on mutexes, if you decide to go this route.)

<a name="footnote-race-condition-terminology"></a>

<div style='font-size: 80%; display: inline;'>* Some may object to the use of the term <i>race condition</i>, since Ethereum does not currently have true parallelism. However, there is still the fundamental feature of logically distinct processes contending for resources, and the same sorts of pitfalls and potential solutions apply.</div>

<a name="transaction-ordering-dependence"></a>

### Transaction-Ordering Dependence (TOD) / Front Running

Above were examples of race conditions involving the attacker executing malicious code *within a single transaction*. The following are a different type of race condition inherent to Blockchains: the fact that *the order of transactions themselves* (within a block) is easily subject to manipulation.

Since a transaction is in the mempool for a short while, one can know what actions will occur, before it is included in a block. This can be troublesome for things like decentralized markets, where a transaction to buy some tokens can be seen, and a market order implemented before the other transaction gets included. Protecting against this is difficult, as it would come down to the specific contract itself. For example, in markets, it would be better to implement batch auctions (this also protects against high frequency trading concerns). Another way to use a pre-commit scheme (“I’m going to submit the details later”).

<a name="timestamp-dependence"></a>

### Timestamp Dependence

Be aware that the timestamp of the block can be manipulated by the miner, and all direct and indirect uses of the timestamp should be considered. *Block numbers* and *average block time* can be used to estimate time, but this is not future proof as block times may change (such as the changes expected during Casper).

```
uint someVariable = now + 1;

if (now % 2 == 0) { // the now can be manipulated by the miner

}

if ((someVariable - 100) % 2 == 0) { // someVariable can be manipulated by the miner

}
```

<a name="integer-overflow-and-underflow"></a>

### Integer Overflow and Underflow

Be aware there are around [20 cases for overflow and underflow](https://github.com/ethereum/solidity/issues/796#issuecomment-253578925).

Consider a simple token transfer:

```
mapping (address => uint256) public balanceOf;

// INSECURE
function transfer(address _to, uint256 _value) {
    /* Check if sender has balance */
    require(balanceOf[msg.sender] > _value);
    /* Add and subtract new balances */
    balanceOf[msg.sender] -= _value;
    balanceOf[_to] += _value;
}

// SECURE
function transfer(address _to, uint256 _value) {
    /* Check if sender has balance and for overflows */
    require(balanceOf[msg.sender] >= _value && balanceOf[_to] + _value >= balanceOf[_to]);

    /* Add and subtract new balances */
    balanceOf[msg.sender] -= _value;
    balanceOf[_to] += _value;
}
```

If a balance reaches the maximum uint value (2^256) it will circle back to zero. This checks for that condition. This may or may not be relevant, depending on the implementation. Think about whether or not the uint value has an opportunity to approach such a large number. Think about how the uint variable changes state, and who has authority to make such changes. If any user can call functions which update the uint value, it's more vulnerable to attack. If only an admin has access to change the variable's state, you might be safe. If a user can increment by only 1 at a time, you are probably also safe because there is no feasible way to reach this limit.

The same is true for underflow. If a uint is made to be less than zero, it will cause an underflow and get set to its maximum value.

Be careful with the smaller data-types like uint8, uint16, uint24...etc: they can even more easily hit their maximum value.

Be aware there are around [20 cases for overflow and underflow](https://github.com/ethereum/solidity/issues/796#issuecomment-253578925).

<a name="dos-with-unexpected-revert"></a>

### DoS with (Unexpected) revert

Consider a simple auction contract:

```
// INSECURE
contract Auction {
    address currentLeader;
    uint highestBid;

    function bid() payable {
        require(msg.value > highestBid);

        require(currentLeader.send(highestBid)); // Refund the old leader, if it fails then revert

        currentLeader = msg.sender;
        highestBid = msg.value;
    }
}
```

When it tries to refund the old leader, it reverts if the refund fails. This means that a malicious bidder can become the leader, while making sure that any refunds to their address will *always* fail. In this way, they can prevent anyone else from calling the `bid()` function, and stay the leader forever. A recommendation is to set up a [pull payment system](https://github.com/ConsenSys/smart-contract-best-practices/#favor-pull-over-push-payments) instead, as described earlier.

Another example is when a contract may iterate through an array to pay users (e.g., supporters in a crowdfunding contract). It's common to want to make sure that each payment succeeds. If not, one should revert. The issue is that if one call fails, you are reverting the whole payout system, meaning the loop will never complete. No one gets paid, because one address is forcing an error.


```
address[] private refundAddresses;
mapping (address => uint) public refunds;

// bad
function refundAll() public {
    for(uint x; x < refundAddresses.length; x++) { // arbitrary length iteration based on how many addresses participated
        require(refundAddresses[x].send(refunds[refundAddresses[x]])) // doubly bad, now a single failure on send will hold up all funds
    }
}
```

Again, the recommended solution is to [favor pull over push payments](#favor-pull-over-push-payments).

<a name="dos-with-block-gas-limit"></a>

### DoS with Block Gas Limit

You may have noticed another problem with the previous example: by paying out to everyone at once, you risk running into the block gas limit. Each Ethereum block can process a certain maximum amount of computation. If you try to go over that, your transaction will fail.

This can lead to problems even in the absence of an intentional attack. However, it's especially bad if an attacker can manipulate the amount of gas needed. In the case of the previous example, the attacker could add a bunch of addresses, each of which needs to get a very small refund. The gas cost of refunding each of the attacker's addresses could therefore end up being more than the gas limit, blocking the refund transaction from happening at all.

This is another reason to [favor pull over push payments](#favor-pull-over-push-payments).

If you absolutely must loop over an array of unknown size, then you should plan for it to potentially take multiple blocks, and therefore require multiple transactions. You will need to keep track of how far you've gone, and be able to resume from that point, as in the following example:

```
struct Payee {
    address addr;
    uint256 value;
}
Payee payees[];
uint256 nextPayeeIndex;

function payOut() {
    uint256 i = nextPayeeIndex;
    while (i < payees.length && msg.gas > 200000) {
      payees[i].addr.send(payees[i].value);
      i++;
    }
    nextPayeeIndex = i;
}
```

You will need to make sure that nothing bad will happen if other transactions are processed while waiting for the next iteration of the `payOut()` function. So only use this pattern if absolutely necessary.

<a name="call-depth-attack"></a>

### ~~Call Depth Attack~~

As of the [EIP 150](https://github.com/ethereum/EIPs/issues/150) hardfork, call depth attacks are no longer relevant<sup><a href='http://ethereum.stackexchange.com/questions/9398/how-does-eip-150-change-the-call-depth-attack'>\*</a></sup> (all gas would be consumed well before reaching the 1024 call depth limit).

<a name="eng-techniques"></a>

## Software Engineering Techniques

As we discussed in the [一般的な知識](#general-philosophy) section, it is not enough to protect yourself against the known attacks. Since the cost of failure on a blockchain can be very high, you must also adapt the way you write software, to account for that risk.

The approach we advocate is to "prepare for failure". It is impossible to know in advance whether your code is secure. However, you can architect your contracts in a way that allows them to fail gracefully, and with minimal damage. This section presents a variety of techniques that will help you prepare for failure.

Note: There's always a risk when you add a new component to your system. A badly designed fail-safe could itself become a vulnerability - as can the interaction between a number of well designed fail-safes. Be thoughtful about each technique you use in your contracts, and consider carefully how they work together to create a robust system.

### Upgrading Broken Contracts

Code will need to be changed if errors are discovered or if improvements need to be made. It is no good to discover a bug, but have no way to deal with it.

Designing an effective upgrade system for smart contracts is an area of active research, and we won't be able to cover all of the complications in this document. However, there are two basic approaches that are most commonly used. The simpler of the two is to have a registry contract that holds the address of the latest version of the contract. A more seamless approach for contract users is to have a contract that forwards calls and data onto the latest version of the contract.

Whatever the technique, it's important to have modularization and good separation between components, so that code changes do not break functionality, orphan data, or require substantial costs to port. In particular, it is usually beneficial to separate complex logic from your data storage, so that you do not have to recreate all of the data in order to change the functionality.

It's also critical to have a secure way for parties to decide to upgrade the code. Depending on your contract, code changes may need to be approved by a single trusted party, a group of members, or a vote of the full set of stakeholders. If this process can take some time, you will want to consider if there are other ways to react more quickly in case of an attack, such as an [emergency stop or circuit-breaker](https://github.com/ConsenSys/smart-contract-best-practices/#circuit-breakers-pause-contract-functionality).

**Example 1: Use a registry contract to store latest version of a contract**

In this example, the calls aren't forwarded, so users should fetch the current address each time before interacting with it.

```
contract SomeRegister {
    address backendContract;
    address[] previousBackends;
    address owner;

    function SomeRegister() {
        owner = msg.sender;
    }

    modifier onlyOwner() {
        require(msg.sender == owner)
        _;
    }

    function changeBackend(address newBackend) public
    onlyOwner()
    returns (bool)
    {
        if(newBackend != backendContract) {
            previousBackends.push(backendContract);
            backendContract = newBackend;
            return true;
        }

        return false;
    }
}
```

There are two main disadvantages to this approach:

1. Users must always look up the current address, and anyone who fails to do so risks using an old version of the contract
2. You will need to think carefully about how to deal with the contract data, when you replace the contract

The alternate approach is to have a contract forward calls and data to the latest version of the contract:

**Example 2: [Use a `DELEGATECALL`](http://ethereum.stackexchange.com/questions/2404/upgradeable-contracts) to forward data and calls**

```
contract Relay {
    address public currentVersion;
    address public owner;

    modifier onlyOwner() {
        require(msg.sender == owner);
        _;
    }

    function Relay(address initAddr) {
        currentVersion = initAddr;
        owner = msg.sender; // this owner may be another contract with multisig, not a single contract owner
    }

    function changeContract(address newVersion) public
    onlyOwner()
    {
        currentVersion = newVersion;
    }

    function() {
        require(currentVersion.delegatecall(msg.data));
    }
}
```

This approach avoids the previous problems, but has problems of its own. You must be extremely careful with how you store data in this contract. If your new contract has a different storage layout than the first, your data may end up corrupted. Additionally, this simple version of the pattern cannot return values from functions, only forward them, which limits its applicability. ([More complex implementations](https://github.com/ownage-ltd/ether-router) attempt to solve this with in-line assembly code and a registry of return sizes.)

Regardless of your approach, it is important to have some way to upgrade your contracts, or they will become unusable when the inevitable bugs are discovered in them.

### Circuit Breakers (Pause contract functionality)

Circuit breakers stop execution if certain conditions are met, and can be useful when new errors are discovered. For example, most actions may be suspended in a contract if a bug is discovered, and the only action now active is a withdrawal. You can either give certain trusted parties the ability to trigger the circuit breaker, or else have programmatic rules that automatically trigger the certain breaker when certain conditions are met.

Example:

```
bool private stopped = false;
address private owner;

modifier isAdmin() {
    require(msg.sender == owner);
    _;
}

function toggleContractActive() isAdmin public
{
    // You can add an additional modifier that restricts stopping a contract to be based on another action, such as a vote of users
    stopped = !stopped;
}

modifier stopInEmergency { if (!stopped) _; }
modifier onlyInEmergency { if (stopped) _; }

function deposit() stopInEmergency public
{
    // some code
}

function withdraw() onlyInEmergency public
{
    // some code
}
```

### Speed Bumps (Delay contract actions)

Speed bumps slow down actions, so that if malicious actions occur, there is time to recover. For example, [The DAO](https://github.com/slockit/DAO/) required 27 days between a successful request to split the DAO and the ability to do so. This ensured the funds were kept within the contract, increasing the likelihood of recovery. In the case of the DAO, there was no effective action that could be taken during the time given by the speed bump, but in combination with our other techniques, they can be quite effective.

Example:

```
struct RequestedWithdrawal {
    uint amount;
    uint time;
}

mapping (address => uint) private balances;
mapping (address => RequestedWithdrawal) private requestedWithdrawals;
uint constant withdrawalWaitPeriod = 28 days; // 4 weeks

function requestWithdrawal() public {
    if (balances[msg.sender] > 0) {
        uint amountToWithdraw = balances[msg.sender];
        balances[msg.sender] = 0; // for simplicity, we withdraw everything;
        // presumably, the deposit function prevents new deposits when withdrawals are in progress

        requestedWithdrawals[msg.sender] = RequestedWithdrawal({
            amount: amountToWithdraw,
            time: now
        });
    }
}

function withdraw() public {
    if(requestedWithdrawals[msg.sender].amount > 0 && now > requestedWithdrawals[msg.sender].time + withdrawalWaitPeriod) {
        uint amountToWithdraw = requestedWithdrawals[msg.sender].amount;
        requestedWithdrawals[msg.sender].amount = 0;

        require(msg.sender.send(amountToWithdraw));
    }
}
```

### Rate Limiting

Rate limiting halts or requires approval for substantial changes. For example, a depositor may only be allowed to withdraw a certain amount or percentage of total deposits over a certain time period (e.g., max 100 ether over 1 day) - additional withdrawals in that time period may fail or require some sort of special approval. Or the rate limit could be at the contract level, with only a certain amount of tokens issued by the contract over a time period.

[Example](https://gist.github.com/PeterBorah/110c331dca7d23236f80e69c83a9d58c#file-circuitbreaker-sol)

<a name="contract-rollout"></a>

### Contract Rollout

Contracts should have a substantial and prolonged testing period - before substantial money is put at risk.

At minimum, you should:

- Have a full test suite with 100% test coverage (or close to it)
- Deploy on your own testnet
- Deploy on the public testnet with substantial testing and bug bounties
- Exhaustive testing should allow various players to interact with the contract at volume
- Deploy on the mainnet in beta, with limits to the amount at risk

##### Automatic Deprecation

During testing, you can force an automatic deprecation by preventing any actions, after a certain time period. For example, an alpha contract may work for several weeks and then automatically shut down all actions, except for the final withdrawal.

```
modifier isActive() {
    require(block.number <= SOME_BLOCK_NUMBER);
    _;
}

function deposit() public
isActive() {
    // some code
}

function withdraw() public {
    // some code
}

```
##### Restrict amount of Ether per user/contract

In the early stages, you can restrict the amount of Ether for any user (or for the entire contract) - reducing the risk.


<a name="bounties"></a>

### Bug Bounty Programs

Some tips for running bounty programs:

- Decide which currency will bounties be distributed in (BTC and/or ETH)
- Decide on an estimated total budget for bounty rewards
- From the budget, determine three tiers of rewards:
  - smallest reward you are willing to give out
  - highest reward that's usually awardable
  - an extra range to be awarded in case of very severe vulnerabilities
- Determine who the bounty judges are (3 may be ideal typically)
- Lead developer should probably be one of the bounty judges
- When a bug report is received, the lead developer, with advice from judges, should evaluate the severity of the bug
- Work at this stage should be in a private repo, and the issue filed on Github
- If it's a bug that should be fixed, in the private repo, a developer should write a test case, which should fail and thus confirm the bug
- Developer should implement the fix and ensure the test now passes; writing additional tests as needed
- Show the bounty hunter the fix; merge the fix back to the public repo is one way
- Determine if bounty hunter has any other feedback about the fix
- Bounty judges determine the size of the reward, based on their evaluation of both the *likelihood* and *impact* of the bug.
- Keep bounty participants informed throughout the process, and then strive to avoid delays in sending them their reward

For an example of the three tiers of rewards, see [Ethereum's Bounty Program](https://bounty.ethereum.org):

> The value of rewards paid out will vary depending on severity of impact. Rewards for minor 'harmless' bugs start at 0.05 BTC. Major bugs, for example leading to consensus issues, will be rewarded up to 5 BTC. Much higher rewards are possible (up to 25 BTC) in case of very severe vulnerabilities.


## Security-related Documentation and Procedures
When launching a contract that will have substantial funds or is required to be mission critical, it is important to include proper documentation. Some documentation related to security includes:

**Specifications and Rollout Plans**

- Specs, diagrams, state machines, models, and other documentation that helps auditors, reviewers, and the community understand what the system is intended to do.
- Many bugs can be found just from the specifications, and they are the least costly to fix.
- Rollout plans that include details listed [here](https://github.com/ConsenSys/smart-contract-best-practices#contract-rollout), and target dates.

**Status**

- Where current code is deployed
- Compiler version, flags used, and steps for verifying the deployed bytecode matches the source code
- Compiler versions and flags that will be used for the different phases of rollout.
- Current status of deployed code (including outstanding issues, performance stats, etc.)

**Known Issues**

- Key risks with contract
  - e.g., You can lose all your money, hacker can vote for certain outcomes
- All known bugs/limitations
- Potential attacks and mitigants
- Potential conflicts of interest (e.g., will be using yourself, like Slock.it did with the DAO)

**History**

- Testing (including usage stats, discovered bugs, length of testing)
- People who have reviewed code (and their key feedback)

**Procedures**

- Action plan in case a bug is discovered (e.g., emergency options, public notification process, etc.)
- Wind down process if something goes wrong (e.g., funders will get percentage of your balance before attack, from remaining funds)
- Responsible disclosure policy (e.g., where to report bugs found, the rules of any bug bounty program)
- Recourse in case of failure (e.g., insurance, penalty fund, no recourse)

**Contact Information**

- Who to contact with issues
- Names of programmers and/or other important parties
- Chat room where questions can be asked

## Security Tools

- [Oyente](https://github.com/melonproject/oyente) - Analyze Ethereum code to find common vulnerabilities, based on this [paper](http://www.comp.nus.edu.sg/~loiluu/papers/oyente.pdf).
- [solidity-coverage](https://github.com/sc-forks/solidity-coverage) - Code coverage for Solidity testing.
- [Solgraph](https://github.com/raineorshine/solgraph) - Generates a DOT graph that visualizes function control flow of a Solidity contract and highlights potential security vulnerabilities.

## Linters

Linters improve code quality by enforcing rules for style and composition, making code easier to read and review.

- [Solium](https://github.com/duaraghav8/Solium) - Yet another Solidity linting.
- [Solint](https://github.com/weifund/solint) - Solidity linting that helps you enforce consistent conventions and avoid errors in your Solidity smart-contracts.
- [Solcheck](https://github.com/federicobond/solcheck) - A linter for Solidity code written in JS and heavily inspired by eslint.



## Future improvements
- **Editor Security Warnings**: Editors will soon alert for common security errors, not just compilation errors. Browser Solidity is getting these features soon.

- **New functional languages that compile to EVM bytecode**: Functional languages gives certain guarantees over procedural languages like Solidity, namely immutability within a function and strong compile time checking. This can reduce the risk of errors by providing deterministic behavior. (for more see [this](https://plus.google.com/u/0/events/cmqejp6d43n5cqkdl3iu0582f4k), Curry-Howard correspondence, and linear logic)

<a name="bibliography"></a>

## Smart Contract Security Bibliography

A lot of this document contains code, examples and insights gained from various parts already written by the community.
Here are some of them.  Feel free to add more.

##### By Ethereum core developers

- [How to Write Safe Smart Contracts](https://chriseth.github.io/notes/talks/safe_solidity) (Christian Reitwiessner)
- [Smart Contract Security](https://blog.ethereum.org/2016/06/10/smart-contract-security/) (Christian Reitwiessner)
- [Thinking about Smart Contract Security](https://blog.ethereum.org/2016/06/19/thinking-smart-contract-security/) (Vitalik Buterin)
- [Solidity](http://solidity.readthedocs.io)
- [Solidity Security Considerations](http://solidity.readthedocs.io/en/latest/security-considerations.html)

##### By Community

- http://forum.ethereum.org/discussion/1317/reentrant-contracts
- http://hackingdistributed.com/2016/06/16/scanning-live-ethereum-contracts-for-bugs/
- http://hackingdistributed.com/2016/06/18/analysis-of-the-dao-exploit/
- http://hackingdistributed.com/2016/06/22/smart-contract-escape-hatches/
- http://martin.swende.se/blog/Devcon1-and-contract-security.html
- http://publications.lib.chalmers.se/records/fulltext/234939/234939.pdf
- http://vessenes.com/deconstructing-thedao-attack-a-brief-code-tour
- http://vessenes.com/ethereum-griefing-wallets-send-w-throw-considered-harmful
- http://vessenes.com/more-ethereum-attacks-race-to-empty-is-the-real-deal
- https://blog.blockstack.org/simple-contracts-are-better-contracts-what-we-can-learn-from-the-dao-6293214bad3a
- https://blog.slock.it/deja-vu-dao-smart-contracts-audit-results-d26bc088e32e
- https://blog.vdice.io/wp-content/uploads/2016/11/vsliceaudit_v1.3.pdf
- https://eprint.iacr.org/2016/1007.pdf
- https://github.com/Bunjin/Rouleth/blob/master/Security.md
- https://github.com/LeastAuthority/ethereum-analyses
- https://medium.com/@ConsenSys/assert-guards-towards-automated-code-bounties-safe-smart-contract-coding-on-ethereum-8e74364b795c
- https://medium.com/@coriacetic/in-bits-we-trust-4e464b418f0b
- https://medium.com/@hrishiolickel/why-smart-contracts-fail-undiscovered-bugs-and-what-we-can-do-about-them-119aa2843007
- https://medium.com/@peterborah/we-need-fault-tolerant-smart-contracts-ec1b56596dbc
- https://medium.com/zeppelin-blog/zeppelin-framework-proposal-and-development-roadmap-fdfa9a3a32ab
- https://pdaian.com/blog/chasing-the-dao-attackers-wake
- http://www.comp.nus.edu.sg/~loiluu/papers/oyente.pdf

## Reviewers

The following people have reviewed this document (date and commit they reviewed in parentheses):
Bill Gleim (07/29/2016 3495fb5)
Bill Gleim (03/15/2017 0244f4e)
-

## License

Licensed under [Apache 2.0](http://www.apache.org/licenses/LICENSE-2.0)

Licensed under [Creative Commons Attribution-NonCommercial-ShareAlike 4.0 International](https://creativecommons.org/licenses/by-nc-sa/4.0/)

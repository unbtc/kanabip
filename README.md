# 概要

Web ページ上で「カスタム文字列（カンマ区切り）」から全組み合わせ（二文字組み合わせ）を作成し、それぞれに対して BIP39 単語リストからランダムに単語を割り当てる仕組みを実装しています。また、ユーザーによる入力の補助（入力候補の提示や入力変更時の入れ替え処理）、並び替え、印刷用のバージョン生成などの機能も含まれています。

## BrainWalletの問題点

BrainWallet（ブレインウォレット）とは、ビットコイン等の暗号通貨の秘密鍵やニーモニック・ワードを脳内に記憶するための方法の一つです。しかし、これには脆弱性があります。私たち人間はパスフレーズに何を使うか予測可能であり、辞書攻撃等によるハッキング技術は日々向上しています。パスワードの大規模なデータベースがいくつか流出しているので、これらをすべてハッシュ化し対応するアドレスに残高があるかどうかを確認することは非常に簡単です。同じ理由で、日本語のBIP39ワードリストから意味のある文章を作ったり、brain2bipなどの平文からBIP39ニーモニックを導き出すツールも推奨されません。

## このツールの目的

簡単に言えば、このツールの目的は「冗長化と記憶支援」です。鍵となる合言葉（24または48文字のひらがな平文）から独自の暗号表を作成し、辞書攻撃からあなたの大事なビットコインを守ります。また、ニーモニックをメモした物理的な財布を紛失することを防ぎます。自分専用の暗号表と合言葉を作成したら、それを無くさないよう大事に保管してください。

## 主な機能と処理の流れ

### 初期化処理（window.onload）
ページ読み込み時に以下の処理を実行します：
- ページ上の要素 `id="date"` に現在日時を表示。
- `updateCount()` を呼び出してカスタム文字の個数を更新。
- `generate()` を呼び出して、カスタム文字の全組み合わせに対し BIP39 単語のマッピングを生成。
- `putSuggest()` を呼び出して、BIP39 単語の候補リストを `<datalist>` 要素に設定。

### BIP39 単語リストの定義
変数 `bip39_words` は、カンマ区切りのBIP39ワードリスト（例："abandon,...,zone,zoo"）を分割して配列に変換しています。

### 入力候補リストの生成（putSuggest 関数）
`<datalist id="bip39_suggest">` の中身を一度空にし、BIP39 単語配列の各単語を `<option value="単語">` として追加します。これにより、入力フィールドで「候補表示（オートコンプリート）」が利用できるようになります。

### 文字の組み合わせ生成（getCombinations 関数）
引数で与えた文字配列 `chars` に対して、2重ループを用いて全ての 2 文字の組み合わせ（例："a" と "b" → "ab"）を生成し、結果の配列を返します。

### 配列のシャッフル（shuffleArray 関数）
引数が配列であるか確認後、Fisher–Yates シャッフルアルゴリズムの改良版として、暗号学的乱数生成関数 `crypto.getRandomValues` を使用してシャッフルします。入力配列のコピーを作成し、元の配列を改変せずにシャッフル済みの新配列を返します。

### 重複チェック（hasDuplicates 関数）
配列内の要素数と `Set` によるユニークな要素の数を比較し、重複があるかどうかを判定します。

### マッピングの生成（generate 関数）
1. **入力値取得と重複チェック**：
    - ユーザー入力欄（`id="chars"`）からカンマ区切りの文字群を取得し、重複がある場合はアラートを表示して処理を中断します。
    
2. **カスタムワードの生成**：
    - 取得した文字群に対して `getCombinations` を呼び出し、全ての 2 文字組み合わせ（カスタムワード）を作成します。

3. **BIP39 単語リストの拡張**：
    - 初めに BIP39 単語リスト全体を `shuffleArray` でシャッフルし、カスタムワード数に満たない場合は再度シャッフルした単語を足し合わせ、必要な個数を確保します。
    - 最後に全体を再度シャッフルすることで、重複単語の割り当てが偏らないようにしています。

4. **マッピングの割り当て**：
    - カスタムワード配列の各要素に対して、拡張した BIP39 単語リストの対応するインデックスの単語をマッピング（連想配列 `mappings`）として設定します。

5. **結果表示**：
    - マッピング数を `id="size-display"` に表示し、`updateDisplay()` を呼び出して画面上に一覧表示します。

### マッピング結果の表示（updateDisplay 関数）
`<ul id="mappings">` の中身を一度クリアし、各マッピング（カスタムワードと対応する BIP39 単語）を `<li>` 要素として追加します。各リスト項目には、カスタムワードのテキストと、編集可能な `<input>`（オートコンプリート有、候補リストは先ほど設定した `bip39_suggest`）が含まれ、入力値が変更された際には `handleInputChange` が呼ばれるよう設定しています。

### 入力変更時の処理（handleInputChange 関数）
ユーザーが入力フィールドの値を変更した場合、新しい単語が BIP39 単語リストに含まれているか確認します。含まれていなければアラートを出し、元の単語に戻します。もし新しい単語がすでに他のカスタムワードに割り当てられている場合は、そのマッピングと入れ替える（スワップ）処理を行い、全体のマッピングの一意性を保ちます。変更後、`updateDisplay` により再描画します。

### その他の補助処理
- `updateCount()`: ユーザーが入力したカスタム文字の個数をカウントして `id="cnt"` に表示します。
- `window.addEventListener('beforeunload', ...)`: ページを離れる直前に全ての `<input type="text">` の値をクリアし、情報の残留を防ぎます。

### 印刷用バージョンの生成（generatePrintVersion 関数）
新規ウィンドウを開いて印刷用の HTML を生成します。元のページの `<style>` 要素を取得して印刷用ページに適用し、マッピング結果をリストとして追加します。


# 使い方

1. **紙と鉛筆の用意**：
   最初に紙と鉛筆を用意してください。PCやスマホは使わず、監視カメラにも気をつけてください。24文字の合言葉を作る場合、例えば「私にはあなたを説得する時間はありません」という文章をひらがなにして2文字ずつに分けます。

>わた　しに　はあ　なた　をせ　つと くす　るじ　かん　'はあ'　りま　せん

もし重複しているペアがあったら、合言葉を考え直します。

>わた　しに　はあ　なた　をせ　つと くす　るじ　かん　'があ'　りま　せん
   
2. **BIP39ニーモニックの記入**：
合言葉の隣に、あなたが持っているBIP39ニーモニックを上から順に書き込んでください。
デフォルトでは濁音や小文字は清音に読み替えています。（例：「るじ」→「るし」、「があ」→「かあ」など）。2つの文字が混在しないように注意してください（例：「しよ」と「しょ」など）。

3. **暗号表の順番決定**：
暗号表に打ち込む順番を決めます。**キーロガー等のマルウェアからニーモニックを守るために、順番はランダムかつダミーとなるワード(1～3単語、合計40単語程度)を間に挟んで入力する必要があります**。組み合わせの隣に適当な順番で12までの数字を書きます。

4. **ツールの使用**：
このツールをシークレットモードで開き、書いた数字の順番にワードを入力していきます。例ではまず最初に「なた」の箇所に"grape"を入力します。単語の途中の文字を入れても自動的にサジェストされます。ダミーを2〜5つほど変則的に加えながら、3)で書いた数字の順に入力します。キーボードは極力使わずに候補から選択してください。

5. **PDF出力**：
12単語すべて入力し終わったら、プリントボタンからPDFに出力して保存します（リロードするとデータが消えてしまうので注意してください）。印刷の向きや倍率を変えることで列の数を整えることができます。可読性に気をつけて大きさを調整してください。PDFデータはJPEG等の画像にも書き出して保存しておくことをお勧めします。

6. **暗号表のテスト**：
暗号表ができたら、それが読めるかどうかテストしてみてください。合言葉を思い出し、隣にある単語を探してください。*ウォレットが回復することを必ず確認し*、この作業に使ったメモは細切れにして厳重に処分してください。

#### 重複単語が入れ替わる
例えばappleと入力された場所にbookと入力した場合、元からbookと入力されていた場所の文字が自動的にappleに入れ替わります。この変更によりワードはデフォルトでは2048種類必ず存在し、セキュリティが強化されます。

## 利点と注意点

### 利点
- 暗号表はコピーして様々な場所に保存可能です。USBメモリ、CD-R、紙、ProtonmailなどのE2Eメールサービス等を活用して紛失を避けてください。
- 冗長化により、同時に失う可能性がない環境を構築できます。

### 注意点
- E2E暗号化がされていないストレージは避ける。
- Googleなどのストレージを使う場合は何らかの方法で暗号化する。
- 同じニーモニック×同じ合言葉の別の暗号表を2つ以上作らないこと。特に同じシードを同じ合言葉で示す別の暗号表があると、入力した単語の集合を推測されるリスクがあります。

暗号表がハッカーに盗まれると辞書攻撃が可能になるため、信頼できる人と共有することが重要です。また、合言葉そのものをインターネットやパソコンに保存しないようにしてください。



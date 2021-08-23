# Japanese-BPEEncoder V2



日本語文字列（UTF-8）のEncoder / Tokenizerです。UTF-8を文字ペア単位で分かち書きし、製数列にエンコードします。



## 日本語版BPEEncoder



機能的には、分かち書きを行うTokenizerに近いものですが、BERTやGPT-2等の機械学習モデルにおける用語に合わせて、Encoderとしています。

実際は、TokenizerとEncoerを組み合わせたもので、日本語文字列の分かち書きと整数列へのマッピングを同時に行います。

※BERTのエンコーダーがBytePairEncoderなのでBPEですが、日本語1文字は1バイトでないので、V2ではクラス名をSubWordEncoder（SWE）としました。



**分かち書き**＋**エンコード**

```
今日は日曜焼き肉定食をたべる　→　[668, 26650, 4281, 28350, 26631, 27447, 12771, 26694, 18147, 26662]
```



### なぜ形態素解析では駄目なのか



従来の自然言語処理では、係り受け解析による形態素毎に文章を分割していました。



**形態素解析**

```
すもももももももものうち　→　すもも|も|もも|も|もも|の|うち
```



これが絶対に駄目という訳では無いのですが、係り受け解析は、基本的に、助詞や名詞のデータベースに基づく、人が作成したルールベースの処理によって、文章を解析します。

BERTやGPT-2等の現代的な機械学習モデルで使用しているSelf-Attentionが強力なのは、そうした人が作成したルールに依存せずに、広域的な単語（Encode後の単位での‘単語’）間の関連性を学習出来るためです。

**要するに、その係り受けを学習したい**のに、肝心の単語の分割が、人が作成したルールベースの処理であっては、人の性能を超えることが出来ない、というのが、形態素解析ではなくもっと細かい単位での学習を行うモチベーションとなります。

また、係り受け解析では、必要となる語彙数が予め解らない点も、機械学習モデルのトレーニングには不向きな理由です。トレーニングデータとテストデータで語彙数の差がある場合など、どうしても&lt;unk&gt;が登場します。



### BPEEncoderとは



そのようなわけで、単語をさらに細かく分割し、かつ&lt;unk&gt;が登場しないですむような小さい単位での学習を行うわけですが、小さい単位と言っても、どの程度まで細かくすれば良いか、によって、Encodeの仕方が変わります。

英語文字圏では、アルファベットは26種類しかありませんから、アルファベット単位での学習での学習を行うと、出力チャンネル数が少なくなりすぎます。

そこで現在の機械学習モデルでは、バイト単位での学習とワード単位での学習の中間となる、サブワード単位での学習を行うことが一般的です（[こちら](https://ai-scholar.tech/articles/others/roberta-ai-230)の記事などが詳しいです）。

オリジナルのBERTやGPT-2等では、 **Byte-Pair Encoding (BPE)** が使われます。



**Byte-Pair Encoding (BPE)**

```
Forschungsinstitute　→　Fo|rs|ch|un|gs|in|st|it|ut|io|ne|n
```



BPEは、まずAscii文字を全て語彙として登録し、トークン中のよく出てくる組み合わせを語彙に追加する、という手順で構築されます。

さらに、英語文字圏では、単語と単語の間にスペースが入るため、単語を分割した際の二番目以降のサブワードは「##」から始まるようにして区別します。

これは要するに、単語内の‘繋がりやすい’アルファベットの組み合わせをサブワードとするものです。



### 日本語版BPEの作成



しかし日本語でそれをやろうとすると、難しいことになってしまいます。

まず、元々の語彙数が、アルファベットよりも段違いに多いので、そのペアを探して登録して、とやってゆくと、語彙数が逆に大きすぎるものになってしまいます。

そのため、例えばBERTでは英語以外のUnicode文字を無理矢理Multilingualモデルに分けており、日本語では本来的には不適切なEncodeがなされてしまうため、**適切なEncoderを作る必要がある**のです。

また、日本語には単語間に区切りがないにもかかわらず、単語の途中に登場するトークンを「##～」と区別してしまうと、最初の単語分かち書きが全体の性能を規定してしまいます。どうせ単語の区切り文字は最終的に消えてしまうのだから、形態素解析はなくても良いのです。

そこで、適切なEncoderの要件として、



- 機械学習モデルの出力とするのにちょうど良い語彙数
- &lt;unk&gt;を不要とする全バイトエンコード
- 2バイト文字3バイト文字といったUTFコードに依存しない
- バイト列と形態素解析の中間となるサブワード単位での分割



という要素を設定し、BPEEncoderを作成しました。



**BPEによる分かち書き**

```
今日は日曜焼き肉定食を食べる　→　今日|は|日曜|焼|き|肉|定食|を|たべ|る
```

V1では「をた|べる」となっていた箇所が、「を|たべ|る」とより適切になっていることが解ります。



### 解説



V1ではSentencePieceで分割した単語をトークンの元としていましたが、V2ではSentencePieceは使わず、コーパスから直接、繋がる頻度の高いペアが作成されました。それに伴い、BPEの区切りも、SentencePieceの単語とは関係なしに、文字種のみに依存するようになりました。

BPEは、漢字・ひらがな・カタカナ毎に作成し、漢字＋ひらがな、ひらがな＋カタカナのような、文字種を又ぐBPEは存在しません。日本語では文字の組み合わせが膨大になるため、BEPの長さは、ひらがな・カタカナは3、漢字は2がMAXになるようにしました。

さらに、V1には無かった、異字体対応もしています。これにより、「慶応」「𢙎応」「慶應」が全て同じIDにエンコードされます（異字体の問題は中々複雑なので完璧ではないかもしれません）。「ゐ」や「ゑ」のような旧字体、「①」「⒉」のような異数字、「㊑」「㋾」のような囲み文字にも対応しています。

```
慶応𢙎応慶應　→　[17749, 17749, 17749] # encode
[17749, 17749, 17749]　→　'慶応慶応慶応' # decode
```

絵文字顔文字は特殊タグとして12種類にまとめています。罫線、2バイト文字の記号、3バイト文字の記号（U2000～U2BFF）は別の特殊タグになります。

最後に、&lt;unk&gt;を避けるために、バイト単位でのエンコードのためのタグを追加し、全語彙が完成します。

全語彙は、32Kトークン、24Kトークン、16Kトークン、8Kトークンの種類を作成し、選べるようにしました。また、BPEを含まない、SingleCharactorEncode用の語彙ファイル(5.6Kトークン)も用意しました。



### 使い方



SWEEncoder_jaクラスを作成して、encode又はdecodeを呼び出します。



```python
from encode_swe import SWEEncoder_ja
import json

with open('ja-swe32k.txt') as f:
    bpe = f.read().split('\n')

with open('emoji.json') as f:
    emoji = json.loads(f.read())

enc = SWEEncoder_ja(bpe, emoji)

p = enc.encode('今日は日曜焼き肉定食をたべる')
print(p)
# [668, 26650, 4281, 28350, 26631, 27447, 12771, 26694, 18147, 26662]

print(enc.decode(p))
# 今日は日曜焼き肉定食をたべる

print([enc.decode([i]) for i in p])
# ['今日','は','日曜','焼','き','肉','定食','を','たべ','る']
```

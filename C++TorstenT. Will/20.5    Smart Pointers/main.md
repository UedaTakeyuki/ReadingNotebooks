# 20.5    Smart Pointers

## 20.5.1    “unique_ptr”
どの時点でも一人の owner が所有すべき、つまり最後に owner がラップされた raw ポインターをクリーンアップする場合、最初の選択肢となるのが　unique_ptr。
主に以下の 2 つの use case がある
- unique_ptr が定義されたスコープから離れることがなく、そのため、unique_ptr の lifetime の終了時に関連付けられた raw ポインタがクリーンアップされる場合。
これはすべての unique_ptr (ローカル変数またはクラスのデータメンバーなど) に適用される
- または、ソースからシンクへの一意のパスが存在する場合。たとえば、 unique_ptr は関数の戻り値になることがある。コンパイラーは、コンテンツ (生のポインター) を内部の unique_ptr から外部の unique_ptr に転送します。

[Listing 20.5     unique_ptr as data field, return value, and local variable.](https://godbolt.org/z/aTTejrnWE)

リスト 20.5 はダイアログを表示するウィンドウアプリケーションを非常に単純化した。
ユーザーは 2 つのボタンのうちの 1 つを押すと思われ、プログラムは押されたボタンを数値として返す -ただし、ここではウィンドウを概略的に考えており、プログラムの簡潔さのゆえ実際にポップアップするウインドウは存在しない。
それでも、ほとんどのウィンドウプログラミングインターフェイスで同様のクラス階層が見られる

すべての unique_ptr のデータフィールドは MyDialog クラスに属している (クラスがそれらを所有している) ため、showDialog() 内でダイアログが destroy されると、ダイアログはすべての unique_ptr とともに適切に削除される
これら（の unique_ptr）は順番に、それぞれの生のポインタを所有し、デストラクタの中で delete を使用して、自身が管理するオブジェクトを解放する

createDialog() で興味深いことが起こる。
return 式は、新しい値を作る -　それは一時値とは言え、基本的には無名変数を作成する。
これは、return ステートメントの最後で、 unique_ptr<MyDialog>{new MyDialog{"…"}} がすぐにまた開放されることを意味するだろうか?
Yes、その通り、しかし showDialog() の中でコピーされることなしには最終的に変数 dialog に return する事はない。
私は「コピーされる」と言っただろうか？ああ、それは不可能だ！ unique_ptr はコピーできまない。
コピーできた場合は、2 つの unique_ptr インスタンスが存在し、両者が同じ raw ポインターを管理する事を欲するになる。
それでは「ユニーク」ではないだろう。
したがって、unique_ptr はコピーではなく "move" される。
正確に言うと、それの中身が move される。
これは unique_ptr が新しく作成された一時値 (内部) に管理される生のポインターを unique_ptr<MyDialog> ダイアログ (外部) に転送することを意味する。

showModal() では、より一層興味深い展開になる。
原則として、ここでも createDialog() と同じことが起きる。
ただし、このメソッドは既存の値を移動して返す。
しかし、なぜ単に return btnOk_ と書かないのでしょうか?これは、btnOk_ が一時値ではないためです。第 16 章で見たように、安全に「盗む」ことができるのは、間もなく消えることがわかっているオブジェクトからのみです。これは一時的な値の場合です。ただし、btnOk_ は一時的な値ではなく、データ フィールド、つまり名前を持つオブジェクトです。コンパイラがその生のポインタを黙って受け取るとしたら、それは驚くべきことです。[ 49 ]

さて、ここでは、既存の変数からコンテンツを抽出したいと考えています。つまり、変数が一時値であるかのように移動します。 std::move を使用すると、コンパイラに「はい、コンパイラさん、btnOk_ を一時値として考慮してください」と指示すると、なんと、createDialog() と同じように動作します。つまり、 unique_ptr は戻り値から生のポインタを取得し、それを showDialog() の pressed に転送します。

ここまでは順調です。プログラムは正常に動作し、「OK を押してくれてありがとう」というテキスト出力が表示されます。ただし、絶対にやってはいけないのは、dialog->showModal() を 2 回呼び出すことです。

```
int showDialogAgain() {
    unique_ptr<MyDialog> dialog = createDialog();
    unique_ptr<Button> pressedOne = dialog->showModal();
    unique_ptr<Button> pressedTwo = dialog->showModal(); // image for unicode &#x1f5f2;not twice
    return pressedTwo->id_; // image for unicode &#x1f5f2;Error; likely crash
}  
```

pressedOne の showModal() を呼び出すと dialog の btnOk_ データフィールドが pressedOne 変数に転送される事に気付いているはずだ。
どちらも unique_ptr だが、生のポインター(かつて new で一度作成された Button*)の所有者になれるのは 1 つのみ。
そして、showModal() の最初の呼び出しで pressedOne にはこの Button ポインターが含まれ、dialog.btnOk_ には「空」の値である nullptr が含まれる。
この例では、2 番目の showModal() は引き続き機能します。次に、この nullptr を pressedTwo に転送する。
この nullptr で ->id_ を試行すると、プログラムは (良くても) クラッシュします。

unique_ptr を戻り値として使用して何ができるかについてのこの小さなチュートリアルを失礼。
これは、一意の所有権を持つ unique_ptr には副作用があることを示している。
これらはまさに望まれる副作用であり、コンパイラーは、それ自体の名前を持つ変数またはデータフィールドから中身を盗むという事を思いつかない。 
それを std::move() で助けた。

[“std::move” Itself Does Not Move](./std::move-Itself-Does-Not-Move.md)

データフィールドはすべて unique_ptr。
ここでは、ポインターのない単純なデータフィールドで十分。
ただし、後で MyDialog をポリモーフィックに拡張したい場合もあるだろう。
そのためには、ポインターが必要。
ここでのポリモーフィックとは、たとえば、Textfield クラスから派生した ColorfulTextfield クラスを作成し、そのクラスのインスタンスを unique_ptr<Textfield> txtFirstName_ に配置することを意味する。
txtFirstName_ がポインター (または参照) でない場合、ColorfulTextfield のプロパティは失われる。
この例は第 15 章にある。

unique_ptr の使用に関する経験則を適用できます。例外は以下の規則を示す：
- ほとんどの場合、unique_ptr を参照引数として使用できる。その後、引数は他の参照と同様に動作する。さらに、「未定義状態」に nullptr を渡すこともでき、これは時に有益。


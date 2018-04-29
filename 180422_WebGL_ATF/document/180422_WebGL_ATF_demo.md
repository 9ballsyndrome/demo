WebGLでATF圧縮テクスチャを表示する
ATF（Adobe Texture Format）はAdobeが作成したテクスチャのフォーマットです。ATFを使えば複数の圧縮テクスチャを1つのファイルで扱えるため、**それぞれの圧縮テクスチャ形式に対応したプラットフォームを意識することなく開発できる**メリットがあります。

もともとはStage3Dという、Adobe Flash Player上からGPUを使用して描画できる機能のために作成されたフォーマットですが、データの中身は通常の圧縮テクスチャのためFlashでなくとも利用できます。

今回はこのATFテクスチャをWebGLで表示してみます。

[![180422_WebGL_ATF_demo.png](https://qiita-image-store.s3.amazonaws.com/0/2679/7cb4649b-34fe-e906-ebf6-92921b8345fe.png)](https://9ballsyndrome.github.io/demo/180422_WebGL_ATF/src/)

- [デモを別ウインドウで再生する](https://9ballsyndrome.github.io/demo/180422_WebGL_ATF/src/)
- [ソースコード](https://github.com/9ballsyndrome/demo/tree/master/180422_WebGL_ATF)

▲ Windows、macOS、iOS、Androidそれぞれの環境において同一のATFファイル2種類（非透過、透過）を読み込み、ATF内部に格納されているDXT1/DXT5、PVRTC、ETC1の圧縮テクスチャから使用できる形式を選択して表示するデモです。

#圧縮テクスチャとは
通常、テクスチャ画像にはJPGやPNGが使われることが多いと思いますが、**GPUはこれらの画像フォーマットをデコードできません。**そのため、WebGLでは`texImage2D()`命令時に内部的にRGB（RGBA）形式にCPUでデコードしてからGPUに転送されます。

たとえば、1024px * 1024pxのJPGファイルをテクスチャとして使用する場合、一律`1024 * 1024 * 3（ピクセルあたりのバイト数） = 3MB`の情報にデコードされ、GPUに転送されます。いかに圧縮率の高いJPGで、ファイルシステム上のサイズが小さくてもGPUに展開されると画像サイズに比例した大きな容量となります。

これは、GPUへ転送する際にCPUでのデコード処理と転送コストが高くなること、**VRAM（GPU上のメモリ）を多く使用すること**を意味します。特に使用できるVRAMの少ないモバイル環境ではテクスチャの使用メモリの増大はコンテンツを作る上で枷となることがあります。

そこで対策として出てくるのが圧縮テクスチャです。圧縮テクスチャはJPGやPNGほどの圧縮率はありませんが、**GPUがデコードできるフォーマットです。**圧縮されたデータはそのままGPUへ転送され、そのままVRAMに配置されます。そしてシェーダからのテクスチャフェッチ時に初めてデコードされます。

GPUでテクスチャフェッチ時にデコードされると聞くと遅くなるイメージがあるかもしれませんが、**テクスチャキャッシュを効率よく利用でき、VRAMへのメモリアクセスが削減できるためむしろ非圧縮の形式より高速になるようです。**

圧縮テクスチャはいいことばかりではなく、大きな問題がふたつあります。ひとつは**フォーマットが非可逆圧縮であり、画質が劣化したりノイズが入る**ことです。これについては下記の記事に役立つ資料が公開されています。

>[CEDEC2013『工程の手戻りを最小限に 圧縮テクスチャ(PVRTC・DXTC・ETC)における傾向と対策』発表資料 | OPTPiX Labs Blog](http://www.webtech.co.jp/blog/optpix_labs/format/5857/)

もうひとつの問題は、**GPUごとに対応する圧縮フォーマットが異なり、環境によって使えるフォーマット/使えないフォーマットがバラバラなことです。**

下記に代表的な圧縮テクスチャフォーマットを挙げます。

## DXT1/DXT5
DXTC（DirectX Texture Compression）ないしS3TC（S3 Texture Compression）はS3 Graphicsの開発した圧縮テクスチャ形式で、主にWindowsとmacOSで使用できます。DXT1はアルファチャンネルに対応していませんが、DXT5は対応しています。

## PVRTC
PVRTC（PowerVR Texture Compression）はImagination Technologiesの開発した圧縮テクスチャ形式で、主にiOSで使用できます。アルファチャンネルに対応しています。
※ iPhone 8/Xに搭載されているA11 BionicチップにはImagination Technologies社製でなく、Apple自社製のGPU（iOS GPU family 4）が採用されていますが、PVRTCはサポートされています。
> [Metal Feature Set Tables](https://developer.apple.com/metal/Metal-Feature-Set-Tables.pdf)

## ETC1
ETC（Ericsson Texture Compression）はEricsson Researchの開発した圧縮テクスチャ形式で、主にAndroidで使用できます。アルファチャンネルには対応していません。

## ETC2
ETC1を拡張した圧縮テクスチャ形式で、主にOpenGL ES 3.0以上のiOSとAndroidで使用できます。アルファチャンネルに対応しています。OpenGL ES 3.0では必須の圧縮テクスチャ形式ですが、多くのデスクトップ向けGPUでハードウェアサポートされていないことから派生規格であるWebGL 2.0では拡張機能になりました。

>[WebGL 2.0 Specification](https://www.khronos.org/registry/webgl/specs/latest/2.0/#5.38)

他にも圧縮テクスチャ形式はありますが、今回関係するのはこの4つです。WebGL 2.0の機能であるETC2を除く[^1]と、WebGLでの対応はおおまかに以下のように考えて良いでしょう。

[^1]: `WEBGL_compressed_texture_etc`拡張は[khronosの仕様](https://www.khronos.org/registry/webgl/extensions/WEBGL_compressed_texture_etc/)ではWebGL 1.0でも対応されているように見えます。しかし、手持ちのAndroidで試したところ、webglコンテクストからは拡張を取得できませんでしたが、webgl2コンテクストからは取得できました。iOSではWebGL 2.0に対応したブラウザがないため確認できませんでした。そのためWebGL 2.0の機能としています。

|プラットフォーム  |圧縮テクスチャ形式  |
|---|---|---|
|Windows/macOS  |DXT1/DXT5  |
|iOS  |PVRTC  |
|Android  |ETC1  |

#ATFの特徴
プラットフォームごとに対応する形式が異なるため、マルチプラットフォームで取り回しにくい圧縮テクスチャですが、Adobeはなんと**これらを1つのファイルにまとめました。**もともとAdobe Flash PlayerやAdobe AIRはプラットフォームの差を吸収し、開発者に意識させないつくりであったため、圧縮テクスチャに関してもその考えを踏襲したのでしょう。

DXT1/DXT5、PVRTC、ETC1、ETC2すべてのテクスチャを1つのATFファイルに含められますが、**「モバイルのみの対応でいいためDXT1/DXT5は省きたい」といったケースにも柔軟に対応できます。**詳しくは下記の資料を参照してください。[^2]

[^2]: 該当記事は古いバージョンのATFの説明のため、ETC2についての記載がありません。最新の情報については英語記事 [Introducing compressed textures with the ATF SDK | Adobe Developer Connection](https://www.adobe.com/devnet/archive/flashruntimes/articles/introducing-compressed-textures.html) を参照してください。

> [ATF SDKを使った圧縮テクスチャの使い方入門 | ADC - Adobe Developer Connection](https://www.adobe.com/jp/devnet/socialgaming/articles/introducing-compressed-textures.html)

ATFファイルを作成するにはATFツールを使用します。ATFツールは最新の[Adobe AIR SDK](https://www.adobe.com/devnet/air/air-sdk-download.html)に含まれています。ダウンロードしたAIR SDK直下の`atftools`フォルダにいくつかのツールが提供されていますが、主に使用するのは以下の2つです。

##png2atf
PNG画像からATF圧縮テクスチャを作成するコマンドラインツールです。基本的な使い方は`-c`オプションをつけて実行します。入力するPNG画像のサイズは各辺がPOT（Power of two、2のべき乗）となっている必要があるので注意してください。

```bash:コマンド
png2atf -c -i inputFileName.png -o outputFileName.atf
```


iOS向けにPVRTC形式のみ含めたい場合は`-c`オプションの引数に`p`を指定します。

```bash:コマンド
png2atf -c p -i inputFileName.png -o outputFileName.atf
```


`-c`オプションの引数は`,（カンマ）`で区切ることでATFファイルに含める形式を任意の組み合わせで指定できます。たとえばモバイル（iOS、Android）向けにPVRTC形式とETC1のみ含めたい場合は`-c`オプションに`p,e`を指定します。

```bash:コマンド
png2atf -c p,e -i inputFileName.png -o outputFileName.atf
```


##ATFViewer
ATFファイルを読み込んで中に含められた各形式のテクスチャ画像やミップマップを表示するGUIツールです。圧縮テクスチャの画質の確認に使います。`atftools\docs\Readme.txt`にある通り、起動するには別途QT SDKのインストールが必要です。

ATFツールの詳しい使い方は下記の資料を参照してください。[^3]

[^3]: 同じくETC2の記載がありません。最新の情報については英語記事 [ATF SDK user's guide | Adobe Developer Connection](https://www.adobe.com/devnet/archive/flashruntimes/articles/atf-users-guide.html) を参照してください。

> [Adobe Texture Format（ATF）ツールの使い方 | ADC - Adobe Developer Connection](https://www.adobe.com/jp/devnet/socialgaming/articles/atf-users-guide.html)

#WebGLでの実装
まずはATFファイルをUint8Arrayのバイト列として読み込みます。

```js:JavaScript
fetch(filePath).then((response) =>
{
  return response.arrayBuffer();
}).then((arrayBuffer) =>
{
  const data = new Uint8Array(arrayBuffer);
  // 以下パース処理
});
```

##データのパース
ATFファイルのバイナリフォーマットは下記を参照します。
>[ATF File Format | Adobe Developer Connection](https://www.adobe.com/devnet/archive/flashruntimes/articles/atf-file-format.html)

ATFファイルはおおまかにテクスチャの情報が格納された**ヘッダー部**と実際のテクスチャデータが格納された**データ部**に別れます。

###ヘッダー部
|Field  |Type  |Comment  |
|---|---|---|
|Signature  |U8[3]  |'ATF'固定  |
|Reserved  |U32  |1バイト目：0x00固定<br>1バイト目：0x00固定<br>3バイト目：下位1bitは`-e`オプションの使用有無、残り7bitは`-n`オプションでミップマップを指定した場合のミップマップ数<br>4バイト目：0xFF固定  |
|Version |U8  |ATFファイルフォーマットのバージョン<BR>※ 2018年4月現在のバージョンは3  |
|Length |U32  |Field～Lengthまでを除いたこのATFファイルのバイト数  |
|Cubemap  |UB[1]  |cubeテクスチャかどうか  |
|Format  |UB[7]  |データ部のフォーマット  |
|Log2Width  |U8  |テクスチャの高さのlog2（最大12）<br>ATFテクスチャの各辺はPOTである必要があるため、実質幅の定義に等しい  |
|Log2Height  |U8  |テクスチャの幅のlog2（最大12）  |
|Count  |U8  |テクスチャ1枚毎のミップマップ数（最大13）  |

###データ部
データ部は複数の形式のテクスチャが[バイト数,データ]の順に格納されたブロックがミップマップ数（ヘッダー部のCount）分繰り返されます。
さらにcubeテクスチャの場合、上記が[Left, Right, Bottom, Top, Back, Front]の順に6面分繰り返されます。

ブロックの定義はヘッダー部のFormatによって異なり、たとえばFormat=3（RAW Compressed）の場合は下記の仕様となります。

|Field  |Type  |Comment  |
|---|---|---|
|DXT1ImageDataLength  |U32  |DXT1データのバイト数  |
|DXT1ImageData  |U8[DXT1ImageDataLength]  |DXT1データ  |
|PVRTCImageDataLength  |U32  |PVRTCデータのバイト数  |
|PVRTCImageData  |U8[PVRTCImageDataLength]  |PVRTCデータ  |
|ETC1ImageDataLength  |U32  |ETC1データのバイト数  |
|ETC1ImageData  |U8[ETC1ImageDataLength]  |ETC1データ  |
|ETC2RgbImageDataLength  |U32  |ETC2データのバイト数  |
|ETC2RgbImageData  |U8[ETC2RgbImageDataLength]  |ETC2データ  |

※ ETC2データに関しては今回使用していません。

バイナリフォーマットに従って順にデータを読んでいき、実際のテクスチャデータ（RAW data）をUint8Arrayの配列として取得します。配列の切り出しには`Uint8Array.subarray()`を使えば無駄なメモリのコピーがされません。

```js:JavaScript
// PVRTCImageData U8[PVRTCImageDataLength]
// RAW PVRTC data

// startByteからpvrtcImageDataLength分の長さのデータをUint8Arrayとして切り出す
const pvrtcImageRawData = data.subarray(startByte, startByte + pvrtcImageDataLength);
```

## 拡張機能の取得
WebGLでは圧縮テクスチャの扱いは拡張機能として提供されているため、使用するには`WebGLRenderingContext.getExtension()`で拡張機能を有効にします。たとえばPVRTCの圧縮テクスチャは`WEBGL_compressed_texture_pvrtc`拡張ですが、ブラウザによってはベンダープレフィックス[^4]をつける必要があります。例として2018年4月時点のiOS Safariでは`WEBKIT_WEBGL_compressed_texture_pvrtc`でないと取得できません。

[^4]: `MOZ_WEBGL_compressed_texture_pvrtc`は[Firefox58で廃止されました。](https://developer.mozilla.org/ja/Firefox/Releases/58#Canvas_and_WebGL)

```js:JavaScript
const extPVRTC = (
  context.getExtension("WEBGL_compressed_texture_pvrtc") ||
  context.getExtension("WEBKIT_WEBGL_compressed_texture_pvrtc")
);
```

拡張機能がサポートされている環境では`getExtension()`で拡張機能オブジェクトが返り、それ以降機能が有効になります。サポートされていない場合は`null`が返ります。

拡張機能がサポートされている場合でも、デバイスが本当に対象の圧縮テクスチャフォーマットをサポートしているかを確認する必要があります。**拡張機能を有効にした後に**`WebGLRenderingContext.getParameter(WebGLRenderingContext.COMPRESSED_TEXTURE_FORMATS)`[^5]を呼んで、サポートしている圧縮テクスチャフォーマットのリストを取得します。リスト内にその時点で有効な圧縮テクスチャフォーマットの種類の値が入っているので、使用するフォーマットが該当するか確認します。

たとえばATFのアルファチャンネルなしPVRTCの場合、使用するフォーマットは`COMPRESSED_RGB_PVRTC_4BPPV1_IMG`ですので、前述の拡張機能オブジェクトから値を取得してリスト内にあるかチェックします。

```js:JavaScript
const supportedCompressedTextureFormat = context.getParameter(context.COMPRESSED_TEXTURE_FORMATS);
const supportsPVRTC = supportedCompressedTextureFormat.includes(extPVRTC.COMPRESSED_RGB_PVRTC_4BPPV1_IMG);
```

[^5]: Uint32Arrayが返ります。スクリプト中で要素のチェックに使用している`Uint32Array.includes()`はES2016の機能です。

下記は今回ATFで使用する圧縮テクスチャフォーマットと拡張機能およびWebGLのフォーマット（internalformat）の対応です。

|ATFのフォーマット  |拡張機能  |WebGLのフォーマット（internalformat）  |
|---|---|---|
|DXT1  |WEBGL_compressed_texture_s3tc  |COMPRESSED_RGB_S3TC_DXT1_EXT |
|DXT5  |WEBGL_compressed_texture_s3tc  |COMPRESSED_RGBA_S3TC_DXT5_EXT |
|PVRTC（アルファチャンネルなし）  |WEBGL_compressed_texture_pvrtc  |COMPRESSED_RGB_PVRTC_4BPPV1_IMG  |
|PVRTC（アルファチャンネルあり）  |WEBGL_compressed_texture_pvrtc  |COMPRESSED_RGBA_PVRTC_4BPPV1_IMG  |
|ETC1  |WEBGL_compressed_texture_etc1  |COMPRESSED_RGB_ETC1_WEBGL  |

## テクスチャの作成
テクスチャオブジェクトの観点で見ると、WebGLでは圧縮テクスチャの扱いは非圧縮テクスチャとほぼ同じです。唯一違う点はGPUにテクスチャを転送するときに非圧縮テクスチャでは`WebGLRenderingContext.texImage2D()`を使っていましたが、圧縮テクスチャでは`WebGLRenderingContext.compressedTexImage2D()`を使うことです。

```js:JavaScript
const compressedTexture = context.createTexture();
context.bindTexture(context.TEXTURE_2D, compressedTexture);
context.compressedTexImage2D(context.TEXTURE_2D, 0, extPVRTC.COMPRESSED_RGB_PVRTC_4BPPV1_IMG, 512, 512, 0, pvrtcImageRawData);
```

`WebGLRenderingContext.compressedTexImage2D()`の引数はそれぞれ下記の通りです。

1. target: テクスチャの種類。通常のテクスチャの場合はTEXTURE_2Dを指定。
1. level: ミップマップレベル。ベースのテクスチャの場合は0を指定
1. internalformat: 圧縮テクスチャのフォーマット。前述の通り拡張機能オブジェクトから取得した値。
1. width: テクスチャの幅。ATFの場合、ヘッダー部のLog2Widthから計算する。
1. height: テクスチャの高さ。同上。
1. border: 0固定。
1. pixels: 圧縮テクスチャのRAW data。ATFの場合は上記の通り読み出した値をそのまま使う。

いくつか注意があります。まず、**自動でミップマップを作ってくれる機能`WebGLRenderingContext.generateMipmap()`は圧縮テクスチャには使えません。**ミップマップを使いたい場合は自前でミップマップ画像を用意し、その数だけ`compressedTexImage2D`を第二引数の`level`を指定して転送します。ミップマップを転送しない場合、当たり前ですが`WebGLRenderingContext.texParameteri()`の`TEXTURE_MIN_FILTER`に`LINEAR_MIPMAP_LINEAR`などのミップマップ系フィルタは使えません。ATFの場合は前述のpng2atfでミップマップも作成されるので、読み出したRAW dataを順番に指定すればOKです。

次に、第七引数`pixels`に指定するのはあくまで圧縮テクスチャのRAW dataということです。ファイルのヘッダーなど余計な情報がついていると転送できません。たとえば、DXTCの圧縮テクスチャは一般的にDDSというファイル形式で扱われますが。このDDSにはRAW dataの他にフォーマットやファイルサイズなどのデータが含まれています。WebGLで自前でDDSファイルをテクスチャとして扱うためには、ヘッダー部を読み捨ててRAW dataのみ取り出す必要があります。

>[DDS ファイル リファレンス](https://msdn.microsoft.com/ja-jp/library/cc372287.aspx)

ATFの場合は前述の通り、パースした`DXT1ImageData`がRAW dataとなっているため、そのまま引数に指定できます。ATFの場合もDDSの場合もWebGLで表示するには結局パースが必要ということですね。

一度データを転送さえしてしまえばdraw時のバインドやシェーダ内での扱いは非圧縮テクスチャと全く同じですので省略します。

## アルファチャンネルの対応
DXTCとPVRTCにはそれぞれアルファチャンネルが対応していますが、ETC1にはアルファチャンネルはありません。ATFの場合、png2atfを使ってアルファチャンネル付き画像を変換するとFormat=5（RAW Compressed with Alpha）となり、`ETC1ImageData`と`ETC1AlphaImageData`が埋め込まれます。それぞれRGB用RAW dataとアルファチャンネル用RAW dataとなっているので、圧縮テクスチャを2枚作成してシェーダ内でRGBとAlphaとして使用します。

```glsl:GLSL
precision mediump float;

uniform sampler2D texture;
uniform sampler2D alphaTexture;
varying vec2 vUV;
    
void main(void)
{
  vec4 rgb = texture2D(texture, vUV);
  vec4 alpha = texture2D(alphaTexture, vUV);
  gl_FragColor = vec4(rgb.rgb, alpha.r);
}
```

以上で冒頭のデモが完成します。

# まとめ
ATFはFlash界隈以外ではマイナーなフォーマットだと思いますが、中身は普通の圧縮テクスチャフォーマットをつなぎ合わせただけのものなのでWebGLでも充分使えることがわかりました。圧縮テクスチャ自体は画質の劣化は結構気になるところなのでうまく見せる工夫は必要ですが、**使いこなせれば特にリソースの少ないモバイル対応で力を発揮する**と思います。別にATFを使わなくてもいいのでぜひ試してみてください。

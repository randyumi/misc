# Javaでの日時の扱いまとめ

## Javaで日時を扱う
現在Javaには主に日付時間を扱うライブラリとしてめぼしいものは以下3つ。
  * DateTime / Calendar / DateFormatとか
  * JodaTime
  * JSR310(Java8)

## それぞれの特徴
### DateTime / Calendar / DateFormat
  * Java7までの標準ライブラリ
  * 低機能
  * マルチスレッドな使い方するとすぐ壊れる

### JodaTime
くわしくはこちら http://www.joda.org/joda-time
  * Jodaプロジェクト謹製のOSS
  * Apacheライセンス
  * Java標準の時刻クラスがあまりにイケていないため開発された

### JSR310
この辺のjavadocをご参照ください http://docs.oracle.com/javase/8/docs/api/index.html?java/time/LocalDateTime.html
最近出来た。

## つかってみる
面倒なのでコードはscala風に記します。

### Java7までのやつ
現在時刻をとってみる
```scala
import java.util.Date
new Date() // java.util.Date = Sun Jun 15 12:54:31 JST 2014
```

好きな時刻で初期化
```scala
val (year, month, day) = (1991, 3, 31)
new Date(year, month, day) // 実は@depreated
new Date(0xdeadbeef) // 直接Unixタイムスタンプを指定するのはOKだけど使いづらい

// 以下のようにCalendarクラスを使ってやるのが普通らしい
import java.util.Calendar
val cal = Calendar.getInstance // 現在時刻を保持したGregorianCalendarのインスタンスが取得できる
cal.set(Calendar.YEAR, 1991) // 年だけ設定してみた
cal.set(year, month - 1 , day, 12, 0, 0) // まとめてセットできる
// 何故か月は1月＝０になっている…
cal.getTime // これで好きな時間にセットしたDateが得られる
cal.set(1991, 3, 31, 0, 0) // 3月31日のつもりでセットしたが3は4月を表す
cal.getTime // Wed May 01 12:00:00 JST 1991 になった
```

カレンダーで時刻加減算
```scala
// addメソッドで行う
cal.add(Calendar.DATE, -1) // 日付を1日戻す
cal.getTime // Tue Apr 30 12:00:00 JST 1991
```

日付をフォーマットして出力する
```scala
import java.text.SimpleDateFormat
val date = cal.getTime
val df = new SimpleDateFormat("yyyy/MM/dd")
df.format(date)

date.toString //この処理でも内部的にはSimpleDateFomatが使用されている
```
* このようにクラス変数を書き換えたりするので書き換えるので
  マルチスレッド環境下で実行すると壊れる
  * http://www.geocities.co.jp/Playtown/1245/java/unsafe_simple_date_format.html
* ここに壊れる例があった
  * http://www.geocities.co.jp/Playtown/1245/java/unsafe_simple_date_format.html
### まとめ
 * 時刻を操作するたびにDate <=> Calendar型を行ったり来たり
 * Calendarがイミュータブルみがあり使いづらい
 * SimpleDateFormatがもろく壊れやすい
   * static finalで使い回したみあふれる機能なのに…

### JodaTime
イミュータブルでスレッドセーフな設計となり、
マルチスレッド環境下で走らせても壊れない。

初期化
```scala
import org.joda.time.DateTime
new DateTime() // 現在時刻
new DateTime(0xdeadbeef) // Unixタイムスタンプで初期化
new DateTime(1991, 3, 31, 12, 0, 0) // 年月日時分秒で初期化
```

時刻を設定してみる

人間が見やすい文字列で表現
```scala
date.toString // 1991-03-31T12:00:00.000+09:00
// toStringの引数にパターンを渡すとその形式で整形してくれる。らくちん(ISO8601)
date.toString("yyyy/MM/dd HH:mm:ss") // 1991/03/31 12:00:00 
```
DBから取り出した日付を表示するときなんかに簡単で便利ですね。

時刻の加減算
```scala
date.plusHours(6).toString // 1991-03-31T18:00:00.000+09:00
// minusもあります
date.minusHours(6).toString // 1991-03-31T06:00:00.000+09:00
// 週の加減算もできて来週の日付とかも得られる
date.minusWeeks(1).toString // 1991-04-07T12:00:00.000+09:00
```

### JSR310
基本的には日時を表すクラスには
* LocalDateTime
* OffsetDateTime
* ZonedDateTime
* JapaneseDate
の4つがあるそう
```scala
import java.time._
LocalDateTime.now // 2014-06-19T11:13:33.000 時刻差なし
ZonedDateTime.now // 2014-06-19T11:13:38.021+9:00[Asia/Tokyo]
OffsetDateTime.now // 2014-06-19T00:26:35.624+09:00 夏時間などが考慮されない
val time = LocalDateTime.of(1991, 3, 31) // 日時指定はof
time.plusHours(3) // 1991-03-31T03:00 JodaTimeみたい！
time // 1991-03-31T00:00 イミュータブル！
// java.util.Dateに変換してみた
Date.from(now.toInstant(ZoneId.systemDefault().getRules().getOffset(now)))
// やばい。いみわかんない
```

JapaneseDateなんていう機能がある
```scala
import java.time.chrono.JapaneseDate
val now = JapaneseDate.now // Japanese Heisei 26-06-19
// Heiseiとかなんだかアツイ
c
JapaneseDate.of(1991, 3, 31) // Japanese Heisei 3-03-31 日時指定
JapaneseDate.of(1000, 1, 1) // java.time.DateTimeException: JapaneseDate before Meiji 6 is not supported 残念
```
官公庁向きの機能だろうか


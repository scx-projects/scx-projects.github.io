# SCX Digest

SCX Digest 是一个摘要和校验值工具库。

它提供 `ScxDigest`，用于对 `byte[]`、`String`、`InputStream` 和 `File` 计算摘要值或校验值，并可以直接返回二进制结果或大写十六进制字符串。

这里的 `Digest` 是广义上的“原始数据的压缩性代表”。

因此 SCX Digest 同时包含两类能力：

```text
MessageDigest    例如 MD5、SHA-1、SHA-256、SHA-384、SHA-512
Checksum         例如 CRC32、CRC32C
```

SCX Digest 本身不是加密库，也不是密码存储库。它只是对 JDK 中常用的 `MessageDigest` 和 `Checksum` 做了一层简单封装，让常见摘要和校验计算更容易使用。

当前版本为 `0.0.1`。

## 安装

### Maven

```xml
<dependency>
    <groupId>dev.scx</groupId>
    <artifactId>scx-digest</artifactId>
    <version>0.0.1</version>
</dependency>
```

## 基本概念

SCX Digest 当前只有一个核心类：

```text
ScxDigest    摘要和校验值工具类
```

它提供两类底层入口：

```text
digest(...)       返回 byte[] 或 long
digestHex(...)    返回大写十六进制字符串
```

同时提供常用算法的快捷方法：

```text
md5 / md5Hex
sha1 / sha1Hex
sha256 / sha256Hex
sha384 / sha384Hex
sha512 / sha512Hex
crc32 / crc32Hex
crc32c / crc32cHex
```

可以简单理解为：

```text
MD5 / SHA 系列     使用 MessageDigest，返回 byte[] 或 hex
CRC32 / CRC32C     使用 Checksum，返回 long 或 hex
```

## 快速开始

计算字符串的 MD5：

```java
import dev.scx.digest.ScxDigest;

var hex = ScxDigest.md5Hex("123");

System.out.println(hex);
```

输出：

```text
202CB962AC59075B964B07152D234B70
```

计算 SHA-256：

```java
var hex = ScxDigest.sha256Hex("123");

System.out.println(hex);
```

输出：

```text
A665A45920422F9D417E4867EFDC4FB8A04A1F3FFF1FA07E998E86F7F7A27AE3
```

计算 CRC32：

```java
var hex = ScxDigest.crc32Hex("123");

System.out.println(hex);
```

输出：

```text
884863D2
```

计算文件摘要：

```java
import dev.scx.digest.ScxDigest;

import java.io.File;

var file = new File("./demo.txt");

var sha256 = ScxDigest.sha256Hex(file);

System.out.println(sha256);
```

计算输入流摘要：

```java
try (var inputStream = Files.newInputStream(Path.of("./demo.txt"))) {
    var sha256 = ScxDigest.sha256Hex(inputStream);
    System.out.println(sha256);
}
```

## 支持的输入类型

SCX Digest 中的摘要和校验方法通常支持四种输入：

```text
byte[]
String
InputStream
File
```

例如 SHA-256：

```java
byte[] sha256(byte[] data)

byte[] sha256(String data)

byte[] sha256(InputStream data) throws IOException

byte[] sha256(File data) throws IOException
```

十六进制版本：

```java
String sha256Hex(byte[] data)

String sha256Hex(String data)

String sha256Hex(InputStream data) throws IOException

String sha256Hex(File data) throws IOException
```

CRC32 也是同样的输入类型：

```java
long crc32(byte[] data)

long crc32(String data)

long crc32(InputStream data) throws IOException

long crc32(File data) throws IOException
```

十六进制版本：

```java
String crc32Hex(byte[] data)

String crc32Hex(String data)

String crc32Hex(InputStream data) throws IOException

String crc32Hex(File data) throws IOException
```

## String 编码

所有 `String` 输入都会使用 UTF-8 编码。

例如：

```java
var hex = ScxDigest.sha256Hex("你好");
```

内部等价于：

```java
var bytes = "你好".getBytes(StandardCharsets.UTF_8);

var hex = ScxDigest.sha256Hex(bytes);
```

这意味着：

```text
同一个字符串在不同平台上不会因为默认字符集不同而得到不同摘要
```

## 十六进制格式

所有 `xxxHex(...)` 方法返回的十六进制字符串都是大写格式。

例如：

```java
ScxDigest.md5Hex("123");
```

结果是：

```text
202CB962AC59075B964B07152D234B70
```

不是：

```text
202cb962ac59075b964b07152d234b70
```

CRC32 和 CRC32C 也一样返回大写十六进制。

```java
ScxDigest.crc32Hex("123");
```

结果：

```text
884863D2
```

## MD5

`md5(...)` 用于计算 MD5 摘要。

返回二进制摘要：

```java
byte[] bytes = ScxDigest.md5("123");
```

返回十六进制摘要：

```java
String hex = ScxDigest.md5Hex("123");
```

结果：

```text
202CB962AC59075B964B07152D234B70
```

支持输入：

```java
byte[] md5(byte[] data)

byte[] md5(String data)

byte[] md5(InputStream data) throws IOException

byte[] md5(File data) throws IOException
```

十六进制版本：

```java
String md5Hex(byte[] data)

String md5Hex(String data)

String md5Hex(InputStream data) throws IOException

String md5Hex(File data) throws IOException
```

示例：

```java
import dev.scx.digest.ScxDigest;

var data = "hello world";

var md5 = ScxDigest.md5Hex(data);

System.out.println(md5);
```

## SHA-1

`sha1(...)` 用于计算 SHA-1 摘要。

返回二进制摘要：

```java
byte[] bytes = ScxDigest.sha1("123");
```

返回十六进制摘要：

```java
String hex = ScxDigest.sha1Hex("123");
```

结果：

```text
40BD001563085FC35165329EA1FF5C5ECBDBBEEF
```

支持输入：

```java
byte[] sha1(byte[] data)

byte[] sha1(String data)

byte[] sha1(InputStream data) throws IOException

byte[] sha1(File data) throws IOException
```

十六进制版本：

```java
String sha1Hex(byte[] data)

String sha1Hex(String data)

String sha1Hex(InputStream data) throws IOException

String sha1Hex(File data) throws IOException
```

示例：

```java
import dev.scx.digest.ScxDigest;

var sha1 = ScxDigest.sha1Hex("hello world");

System.out.println(sha1);
```

## SHA-256

`sha256(...)` 用于计算 SHA-256 摘要。

返回二进制摘要：

```java
byte[] bytes = ScxDigest.sha256("123");
```

返回十六进制摘要：

```java
String hex = ScxDigest.sha256Hex("123");
```

结果：

```text
A665A45920422F9D417E4867EFDC4FB8A04A1F3FFF1FA07E998E86F7F7A27AE3
```

支持输入：

```java
byte[] sha256(byte[] data)

byte[] sha256(String data)

byte[] sha256(InputStream data) throws IOException

byte[] sha256(File data) throws IOException
```

十六进制版本：

```java
String sha256Hex(byte[] data)

String sha256Hex(String data)

String sha256Hex(InputStream data) throws IOException

String sha256Hex(File data) throws IOException
```

示例：

```java
import dev.scx.digest.ScxDigest;

var sha256 = ScxDigest.sha256Hex("hello world");

System.out.println(sha256);
```

## SHA-384

`sha384(...)` 用于计算 SHA-384 摘要。

返回二进制摘要：

```java
byte[] bytes = ScxDigest.sha384("123");
```

返回十六进制摘要：

```java
String hex = ScxDigest.sha384Hex("123");
```

结果：

```text
9A0A82F0C0CF31470D7AFFEDE3406CC9AA8410671520B727044EDA15B4C25532A9B5CD8AAF9CEC4919D76255B6BFB00F
```

支持输入：

```java
byte[] sha384(byte[] data)

byte[] sha384(String data)

byte[] sha384(InputStream data) throws IOException

byte[] sha384(File data) throws IOException
```

十六进制版本：

```java
String sha384Hex(byte[] data)

String sha384Hex(String data)

String sha384Hex(InputStream data) throws IOException

String sha384Hex(File data) throws IOException
```

示例：

```java
import dev.scx.digest.ScxDigest;

var sha384 = ScxDigest.sha384Hex("hello world");

System.out.println(sha384);
```

## SHA-512

`sha512(...)` 用于计算 SHA-512 摘要。

返回二进制摘要：

```java
byte[] bytes = ScxDigest.sha512("123");
```

返回十六进制摘要：

```java
String hex = ScxDigest.sha512Hex("123");
```

结果：

```text
3C9909AFEC25354D551DAE21590BB26E38D53F2173B8D3DC3EEE4C047E7AB1C1EB8B85103E3BE7BA613B31BB5C9C36214DC9F14A42FD7A2FDB84856BCA5C44C2
```

支持输入：

```java
byte[] sha512(byte[] data)

byte[] sha512(String data)

byte[] sha512(InputStream data) throws IOException

byte[] sha512(File data) throws IOException
```

十六进制版本：

```java
String sha512Hex(byte[] data)

String sha512Hex(String data)

String sha512Hex(InputStream data) throws IOException

String sha512Hex(File data) throws IOException
```

示例：

```java
import dev.scx.digest.ScxDigest;

var sha512 = ScxDigest.sha512Hex("hello world");

System.out.println(sha512);
```

## CRC32

`crc32(...)` 用于计算 CRC32 校验值。

返回 long：

```java
long value = ScxDigest.crc32("123");
```

返回十六进制字符串：

```java
String hex = ScxDigest.crc32Hex("123");
```

结果：

```text
884863D2
```

支持输入：

```java
long crc32(byte[] data)

long crc32(String data)

long crc32(InputStream data) throws IOException

long crc32(File data) throws IOException
```

十六进制版本：

```java
String crc32Hex(byte[] data)

String crc32Hex(String data)

String crc32Hex(InputStream data) throws IOException

String crc32Hex(File data) throws IOException
```

示例：

```java
import dev.scx.digest.ScxDigest;

var crc32 = ScxDigest.crc32("hello world");

var crc32Hex = ScxDigest.crc32Hex("hello world");

System.out.println(crc32);
System.out.println(crc32Hex);
```

## CRC32C

`crc32c(...)` 用于计算 CRC32C 校验值。

返回 long：

```java
long value = ScxDigest.crc32c("123");
```

返回十六进制字符串：

```java
String hex = ScxDigest.crc32cHex("123");
```

结果：

```text
107B2FB2
```

支持输入：

```java
long crc32c(byte[] data)

long crc32c(String data)

long crc32c(InputStream data) throws IOException

long crc32c(File data) throws IOException
```

十六进制版本：

```java
String crc32cHex(byte[] data)

String crc32cHex(String data)

String crc32cHex(InputStream data) throws IOException

String crc32cHex(File data) throws IOException
```

示例：

```java
import dev.scx.digest.ScxDigest;

var crc32c = ScxDigest.crc32c("hello world");

var crc32cHex = ScxDigest.crc32cHex("hello world");

System.out.println(crc32c);
System.out.println(crc32cHex);
```

## 通用 digest

除了快捷方法，也可以直接使用通用 `digest(...)`。

### MessageDigest 算法

如果想使用任意 JDK 支持的 `MessageDigest` 算法，可以传入算法名。

```java
import dev.scx.digest.ScxDigest;

byte[] bytes = ScxDigest.digest("123", "SHA-256");
```

十六进制版本：

```java
String hex = ScxDigest.digestHex("123", "SHA-256");
```

支持输入：

```java
byte[] digest(byte[] data, String algorithm) throws NoSuchAlgorithmException

byte[] digest(String data, String algorithm) throws NoSuchAlgorithmException

byte[] digest(InputStream data, String algorithm)
        throws IOException, NoSuchAlgorithmException

byte[] digest(File data, String algorithm)
        throws IOException, NoSuchAlgorithmException
```

十六进制版本：

```java
String digestHex(byte[] data, String algorithm) throws NoSuchAlgorithmException

String digestHex(String data, String algorithm) throws NoSuchAlgorithmException

String digestHex(InputStream data, String algorithm)
        throws IOException, NoSuchAlgorithmException

String digestHex(File data, String algorithm)
        throws IOException, NoSuchAlgorithmException
```

示例：

```java
try {
    var hex = ScxDigest.digestHex("hello", "SHA-256");
    System.out.println(hex);
} catch (NoSuchAlgorithmException e) {
    throw new IllegalArgumentException("Unsupported digest algorithm", e);
}
```

如果算法名不存在，会抛出：

```text
NoSuchAlgorithmException
```

### Checksum 算法

如果想使用任意 `Checksum` 实现，可以传入 `Supplier`。

```java
import dev.scx.digest.ScxDigest;

import java.util.zip.CRC32;

long value = ScxDigest.digest("123", CRC32::new);
```

十六进制版本：

```java
String hex = ScxDigest.digestHex("123", CRC32::new);
```

支持输入：

```java
long digest(byte[] data, Supplier checksumSupplier)

long digest(String data, Supplier checksumSupplier)

long digest(InputStream data, Supplier checksumSupplier) throws IOException

long digest(File data, Supplier checksumSupplier) throws IOException
```

十六进制版本：

```java
String digestHex(byte[] data, Supplier checksumSupplier)

String digestHex(String data, Supplier checksumSupplier)

String digestHex(InputStream data, Supplier checksumSupplier) throws IOException

String digestHex(File data, Supplier checksumSupplier) throws IOException
```

示例：

```java
import java.util.zip.Adler32;

var value = ScxDigest.digest("hello", Adler32::new);

var hex = ScxDigest.digestHex("hello", Adler32::new);
```

这适合需要使用 JDK 其它 `Checksum` 实现的场景。

## 二进制结果和十六进制结果

MessageDigest 类方法有两种返回形式：

```text
sha256(...)       byte[]
sha256Hex(...)    String
```

例如：

```java
byte[] bytes = ScxDigest.sha256("123");

String hex = ScxDigest.sha256Hex("123");
```

它们表示的是同一个摘要。

只是一个是原始二进制，一个是大写十六进制字符串。

CRC 类方法也有两种返回形式：

```text
crc32(...)       long
crc32Hex(...)    String
```

例如：

```java
long value = ScxDigest.crc32("123");

String hex = ScxDigest.crc32Hex("123");
```

CRC32 和 CRC32C 本质上是 32 位校验值，所以十六进制结果通常是 8 个字符。

## byte[] 输入

`byte[]` 输入会直接更新摘要或校验对象。

```java
var data = new byte[]{1, 2, 3};

var hex = ScxDigest.sha256Hex(data);
```

内部逻辑可以理解为：

```java
var messageDigest = MessageDigest.getInstance("SHA-256");

messageDigest.update(data);

byte[] result = messageDigest.digest();
```

对于 CRC：

```java
var checksum = new CRC32();

checksum.update(data);

long value = checksum.getValue();
```

需要注意：

```text
byte[] 不会被复制
ScxDigest 不会修改传入的 byte[]
调用期间不要并发修改这个 byte[]
```

## String 输入

`String` 输入会先转换为 UTF-8 字节。

```java
var hex = ScxDigest.sha256Hex("hello");
```

等价于：

```java
var bytes = "hello".getBytes(StandardCharsets.UTF_8);

var hex = ScxDigest.sha256Hex(bytes);
```

因此，如果你已经有明确编码后的字节数组，可以直接传 `byte[]`。

```java
var bytes = text.getBytes(StandardCharsets.UTF_16LE);

var hex = ScxDigest.sha256Hex(bytes);
```

## InputStream 输入

`InputStream` 输入会按块读取，直到 EOF。

```java
try (var inputStream = Files.newInputStream(Path.of("./demo.txt"))) {
    var hex = ScxDigest.sha256Hex(inputStream);
    System.out.println(hex);
}
```

内部读取缓冲区大小是：

```text
64 * 1024
```

也就是：

```text
65536 bytes
```

需要注意：

```text
ScxDigest 不会关闭传入的 InputStream
```

因此调用方应自己管理输入流生命周期。

推荐写法：

```java
try (var inputStream = Files.newInputStream(path)) {
    var hex = ScxDigest.sha256Hex(inputStream);
}
```

调用结束后，`InputStream` 会被读取到 EOF。

如果还需要重新读取，需要重新打开流，或者先把数据缓存起来。

## File 输入

`File` 输入会在内部打开 `FileInputStream`。

```java
var file = new File("./demo.txt");

var hex = ScxDigest.sha256Hex(file);
```

内部会自动关闭打开的 `FileInputStream`。

等价逻辑可以理解为：

```java
try (var inputStream = new FileInputStream(file)) {
    return ScxDigest.sha256Hex(inputStream);
}
```

如果文件不存在、不可读或读取失败，会抛出：

```text
IOException
```

## null 处理

所有输入数据参数都不能为 `null`。

例如：

```java
ScxDigest.sha256Hex((String) null);
```

会抛出：

```text
NullPointerException
```

异常信息类似：

```text
Data must not be null !!!
```

这适用于：

```text
byte[]
String
InputStream
File
```

需要注意，算法名 `algorithm` 和 `checksumSupplier` 也应传入有效值。

否则会由 JDK 或后续调用抛出相应异常。

## 异常

SCX Digest 中常见异常包括：

```text
NullPointerException        数据为 null
NoSuchAlgorithmException    指定的 MessageDigest 算法不存在
IOException                 读取 InputStream 或 File 失败
IllegalStateException       快捷算法理论上不存在时的包装异常
```

### NoSuchAlgorithmException

通用 `digest(..., String algorithm)` 和 `digestHex(..., String algorithm)` 会显式抛出 `NoSuchAlgorithmException`。

示例：

```java
try {
    var hex = ScxDigest.digestHex("hello", "UNKNOWN");
} catch (NoSuchAlgorithmException e) {
    System.out.println("不支持的摘要算法");
}
```

### IOException

`InputStream` 和 `File` 版本会抛出 `IOException`。

```java
try {
    var hex = ScxDigest.sha256Hex(new File("./demo.txt"));
} catch (IOException e) {
    System.out.println("读取失败");
}
```

### IllegalStateException

`md5(...)`、`sha1(...)`、`sha256(...)`、`sha384(...)`、`sha512(...)` 这些快捷方法内部假定对应算法一定存在。

因此它们不会向外显式抛出 `NoSuchAlgorithmException`。

如果极端情况下对应算法不存在，会包装成：

```text
IllegalStateException
```

这让常用算法调用更简单：

```java
var hex = ScxDigest.sha256Hex("hello");
```

不需要写：

```java
try {
    ...
} catch (NoSuchAlgorithmException e) {
    ...
}
```

## 方法总览

### 通用 MessageDigest

```java
byte[] digest(byte[] data, String algorithm)
        throws NoSuchAlgorithmException

byte[] digest(String data, String algorithm)
        throws NoSuchAlgorithmException

byte[] digest(InputStream data, String algorithm)
        throws IOException, NoSuchAlgorithmException

byte[] digest(File data, String algorithm)
        throws IOException, NoSuchAlgorithmException
```

```java
String digestHex(byte[] data, String algorithm)
        throws NoSuchAlgorithmException

String digestHex(String data, String algorithm)
        throws NoSuchAlgorithmException

String digestHex(InputStream data, String algorithm)
        throws IOException, NoSuchAlgorithmException

String digestHex(File data, String algorithm)
        throws IOException, NoSuchAlgorithmException
```

### 通用 Checksum

```java
long digest(byte[] data, Supplier checksumSupplier)

long digest(String data, Supplier checksumSupplier)

long digest(InputStream data, Supplier checksumSupplier)
        throws IOException

long digest(File data, Supplier checksumSupplier)
        throws IOException
```

```java
String digestHex(byte[] data, Supplier checksumSupplier)

String digestHex(String data, Supplier checksumSupplier)

String digestHex(InputStream data, Supplier checksumSupplier)
        throws IOException

String digestHex(File data, Supplier checksumSupplier)
        throws IOException
```

### MD5

```java
byte[] md5(byte[] data)

byte[] md5(String data)

byte[] md5(InputStream data) throws IOException

byte[] md5(File data) throws IOException
```

```java
String md5Hex(byte[] data)

String md5Hex(String data)

String md5Hex(InputStream data) throws IOException

String md5Hex(File data) throws IOException
```

### SHA-1

```java
byte[] sha1(byte[] data)

byte[] sha1(String data)

byte[] sha1(InputStream data) throws IOException

byte[] sha1(File data) throws IOException
```

```java
String sha1Hex(byte[] data)

String sha1Hex(String data)

String sha1Hex(InputStream data) throws IOException

String sha1Hex(File data) throws IOException
```

### SHA-256

```java
byte[] sha256(byte[] data)

byte[] sha256(String data)

byte[] sha256(InputStream data) throws IOException

byte[] sha256(File data) throws IOException
```

```java
String sha256Hex(byte[] data)

String sha256Hex(String data)

String sha256Hex(InputStream data) throws IOException

String sha256Hex(File data) throws IOException
```

### SHA-384

```java
byte[] sha384(byte[] data)

byte[] sha384(String data)

byte[] sha384(InputStream data) throws IOException

byte[] sha384(File data) throws IOException
```

```java
String sha384Hex(byte[] data)

String sha384Hex(String data)

String sha384Hex(InputStream data) throws IOException

String sha384Hex(File data) throws IOException
```

### SHA-512

```java
byte[] sha512(byte[] data)

byte[] sha512(String data)

byte[] sha512(InputStream data) throws IOException

byte[] sha512(File data) throws IOException
```

```java
String sha512Hex(byte[] data)

String sha512Hex(String data)

String sha512Hex(InputStream data) throws IOException

String sha512Hex(File data) throws IOException
```

### CRC32

```java
long crc32(byte[] data)

long crc32(String data)

long crc32(InputStream data) throws IOException

long crc32(File data) throws IOException
```

```java
String crc32Hex(byte[] data)

String crc32Hex(String data)

String crc32Hex(InputStream data) throws IOException

String crc32Hex(File data) throws IOException
```

### CRC32C

```java
long crc32c(byte[] data)

long crc32c(String data)

long crc32c(InputStream data) throws IOException

long crc32c(File data) throws IOException
```

```java
String crc32cHex(byte[] data)

String crc32cHex(String data)

String crc32cHex(InputStream data) throws IOException

String crc32cHex(File data) throws IOException
```

## 完整示例：计算字符串摘要

```java
import dev.scx.digest.ScxDigest;

public class StringDigestDemo {

    public static void main(String[] args) {
        var data = "123";

        var sha1 = ScxDigest.sha1Hex(data);
        var sha256 = ScxDigest.sha256Hex(data);
        var sha384 = ScxDigest.sha384Hex(data);
        var sha512 = ScxDigest.sha512Hex(data);
        var md5 = ScxDigest.md5Hex(data);
        var crc32 = ScxDigest.crc32Hex(data);
        var crc32c = ScxDigest.crc32cHex(data);

        System.out.println(sha1);
        System.out.println(sha256);
        System.out.println(sha384);
        System.out.println(sha512);
        System.out.println(md5);
        System.out.println(crc32);
        System.out.println(crc32c);
    }

}
```

输出：

```text
40BD001563085FC35165329EA1FF5C5ECBDBBEEF
A665A45920422F9D417E4867EFDC4FB8A04A1F3FFF1FA07E998E86F7F7A27AE3
9A0A82F0C0CF31470D7AFFEDE3406CC9AA8410671520B727044EDA15B4C25532A9B5CD8AAF9CEC4919D76255B6BFB00F
3C9909AFEC25354D551DAE21590BB26E38D53F2173B8D3DC3EEE4C047E7AB1C1EB8B85103E3BE7BA613B31BB5C9C36214DC9F14A42FD7A2FDB84856BCA5C44C2
202CB962AC59075B964B07152D234B70
884863D2
107B2FB2
```

## 完整示例：计算文件 SHA-256

```java
import dev.scx.digest.ScxDigest;

import java.io.File;

public class FileDigestDemo {

    public static void main(String[] args) throws Exception {
        var file = new File("./demo.txt");

        var sha256 = ScxDigest.sha256Hex(file);

        System.out.println(sha256);
    }

}
```

## 完整示例：计算 InputStream 摘要

```java
import dev.scx.digest.ScxDigest;

import java.nio.file.Files;
import java.nio.file.Path;

public class InputStreamDigestDemo {

    public static void main(String[] args) throws Exception {
        var path = Path.of("./demo.txt");

        try (var inputStream = Files.newInputStream(path)) {
            var md5 = ScxDigest.md5Hex(inputStream);

            System.out.println(md5);
        }
    }

}
```

需要注意，`ScxDigest.md5Hex(inputStream)` 会把这个流读取到 EOF。

如果之后还要再次读取同一个文件，应重新打开一个新的 `InputStream`。

## 完整示例：使用自定义 Checksum

```java
import dev.scx.digest.ScxDigest;

import java.util.zip.Adler32;

public class CustomChecksumDemo {

    public static void main(String[] args) {
        var data = "hello world";

        var value = ScxDigest.digest(data, Adler32::new);

        var hex = ScxDigest.digestHex(data, Adler32::new);

        System.out.println(value);
        System.out.println(hex);
    }

}
```

## 完整示例：比较摘要

```java
import dev.scx.digest.ScxDigest;

import java.io.File;

public class CompareDigestDemo {

    public static void main(String[] args) throws Exception {
        var file1 = new File("./a.txt");
        var file2 = new File("./b.txt");

        var sha2561 = ScxDigest.sha256Hex(file1);
        var sha2562 = ScxDigest.sha256Hex(file2);

        if (sha2561.equals(sha2562)) {
            System.out.println("文件内容相同");
        } else {
            System.out.println("文件内容不同");
        }
    }

}
```

## 完整示例：校验下载文件

```java
import dev.scx.digest.ScxDigest;

import java.io.File;

public class VerifyFileDemo {

    public static void main(String[] args) throws Exception {
        var file = new File("./download.zip");

        var expected = "A665A45920422F9D417E4867EFDC4FB8A04A1F3FFF1FA07E998E86F7F7A27AE3";

        var actual = ScxDigest.sha256Hex(file);

        if (!expected.equals(actual)) {
            throw new IllegalStateException("文件校验失败");
        }

        System.out.println("文件校验通过");
    }

}
```

## 设计说明

### 1. ScxDigest 是纯静态工具类

`ScxDigest` 是一个工具类。

所有能力都通过静态方法提供。

```java
ScxDigest.sha256Hex(...)

ScxDigest.md5Hex(...)

ScxDigest.crc32(...)
```

通常不需要创建 `ScxDigest` 实例。

### 2. Digest 在这里是广义概念

在 JDK 中，`MessageDigest` 和 `Checksum` 是两套不同接口。

SCX Digest 把它们放在同一个工具类中，是因为它们都可以看作是：

```text
对原始数据计算出的压缩性代表值
```

所以：

```text
SHA-256 是 digest
CRC32 也是广义 digest
```

但它们用途不同。

```text
SHA / MD5 系列    更常用于摘要、指纹、内容标识
CRC 系列          更常用于错误检测、传输校验、快速校验
```

### 3. 快捷方法不暴露 NoSuchAlgorithmException

像下面这些算法是固定字符串：

```text
MD5
SHA-1
SHA-256
SHA-384
SHA-512
```

快捷方法内部假定它们存在。

因此：

```java
ScxDigest.sha256Hex("hello")
```

不会要求调用方捕获 `NoSuchAlgorithmException`。

如果你使用的是通用算法名入口：

```java
ScxDigest.digestHex("hello", algorithm)
```

则需要处理 `NoSuchAlgorithmException`。

### 4. 十六进制统一大写

SCX Digest 使用统一的大写十六进制格式。

这样更适合：

```text
日志输出
文件校验值展示
和常见 checksum 文件比较
大小写敏感字符串比较
```

如果你需要小写，可以在调用方转换：

```java
var lower = ScxDigest.sha256Hex("hello").toLowerCase();
```

### 5. String 固定使用 UTF-8

`String` 方法固定使用 UTF-8，可以避免平台默认字符集差异。

如果你需要其它编码，应先自己转成 `byte[]`。

```java
var bytes = text.getBytes(StandardCharsets.UTF_16LE);

var hex = ScxDigest.sha256Hex(bytes);
```

### 6. InputStream 不由 ScxDigest 关闭

`InputStream` 版本只负责读取，不负责关闭。

这样设计可以避免工具方法偷偷关闭调用方仍然需要使用的流。

如果你希望自动关闭，使用 try-with-resources：

```java
try (var in = Files.newInputStream(path)) {
    return ScxDigest.sha256Hex(in);
}
```

### 7. File 版本会自动关闭内部流

`File` 版本内部会自己打开 `FileInputStream`，因此也会自己关闭这个内部流。

调用方只需要传入 `File`。

```java
var hex = ScxDigest.sha256Hex(file);
```

### 8. 缓冲区大小固定为 64KB

处理 `InputStream` 和 `File` 时，内部使用固定缓冲区读取。

```text
64KB
```

这避免了一次性把整个文件读入内存。

因此文件版本适合处理较大的文件。

### 9. 这个库不负责密码哈希

SCX Digest 只是普通摘要工具。

它不会做：

```text
加盐
多轮迭代
内存困难计算
密码哈希参数管理
```

如果要存储密码，应使用专门的密码哈希方案，而不是直接使用 `md5Hex(...)` 或 `sha256Hex(...)`。

### 10. MD5 / SHA-1 保留是为了兼容场景

`md5(...)` 和 `sha1(...)` 仍然很常见。

例如：

```text
旧系统兼容
文件名指纹
简单内容标识
第三方接口要求
历史数据校验
```

但不要把它们当作现代安全场景下的强安全摘要选择。

如果只是做普通文件内容指纹，通常可以优先使用：

```text
SHA-256
```

## 常见问题

### SCX Digest 是加密库吗？

不是。

它只负责摘要和校验值计算。

摘要不是加密。

加密通常可以解密，摘要通常不能还原原文。

### digest 和 checksum 有什么区别？

在 JDK 中：

```text
MessageDigest    例如 MD5、SHA-256
Checksum         例如 CRC32、CRC32C
```

它们是不同接口。

SCX Digest 把它们放在同一个工具类中，是为了方便使用。

### Hex 返回值是大写还是小写？

大写。

例如：

```text
202CB962AC59075B964B07152D234B70
```

### String 使用什么编码？

UTF-8。

### InputStream 方法会关闭流吗？

不会。

调用方需要自己关闭。

```java
try (var in = Files.newInputStream(path)) {
    var hex = ScxDigest.sha256Hex(in);
}
```

### File 方法会关闭流吗？

会。

File 版本内部会打开并关闭自己的 `FileInputStream`。

### 文件很大时会一次性读入内存吗？

不会。

`File` 和 `InputStream` 版本会使用缓冲区分块读取。

缓冲区大小是：

```text
64KB
```

### null 数据会返回空摘要吗？

不会。

如果数据参数为 `null`，会抛出 `NullPointerException`。

### 空字符串可以计算吗？

可以。

```java
var hex = ScxDigest.sha256Hex("");
```

空字符串不是 `null`，它会按 UTF-8 转为长度为 `0` 的 byte[] 后计算摘要。

### 空 byte[] 可以计算吗？

可以。

```java
var hex = ScxDigest.sha256Hex(new byte[]{});
```

### 算法名写错会怎样？

如果使用通用方法：

```java
ScxDigest.digestHex("hello", "UNKNOWN");
```

会抛出：

```text
NoSuchAlgorithmException
```

### 快捷方法为什么不抛 NoSuchAlgorithmException？

因为快捷方法使用的是固定算法名。

例如：

```text
SHA-256
MD5
SHA-512
```

这些方法内部假定算法存在。

如果极端情况下不存在，会包装成 `IllegalStateException`。

### CRC32 返回 long，为什么 hex 是 8 位？

CRC32 是 32 位校验值。

`crc32Hex(...)` 会把结果格式化为 8 位十六进制字符串。

### 可以使用 Adler32 吗？

可以使用通用 Checksum 入口。

```java
var hex = ScxDigest.digestHex("hello", Adler32::new);
```

### 可以使用 SHA3 吗？

如果当前 JDK 的 `MessageDigest` 支持对应算法名，就可以使用通用入口。

```java
var hex = ScxDigest.digestHex("hello", "SHA3-256");
```

如果不支持，会抛出 `NoSuchAlgorithmException`。

### 二进制摘要怎么转成 hex？

如果你需要 hex，优先直接使用 `xxxHex(...)`。

```java
var hex = ScxDigest.sha256Hex(data);
```

如果已经拿到了 `byte[]`，可以自行使用 JDK `HexFormat` 转换。

### 可以用 md5Hex 存密码吗？

不建议。

SCX Digest 不提供密码哈希所需的加盐、迭代和安全参数管理能力。

### 什么时候用 SHA-256？

适合常见的内容指纹、文件校验、数据摘要和唯一性比较场景。

```java
var hex = ScxDigest.sha256Hex(file);
```

### 什么时候用 CRC32？

适合快速校验、传输错误检测或和已有 CRC32 协议兼容的场景。

```java
var hex = ScxDigest.crc32Hex(data);
```

### 什么时候用 digest(..., String algorithm)？

当你需要使用快捷方法没有提供的 `MessageDigest` 算法时。

```java
var hex = ScxDigest.digestHex(data, "SHA-512/256");
```

前提是当前 JDK 支持这个算法名。

### 什么时候用 digest(..., Supplier checksumSupplier)？

当你需要使用 CRC32 / CRC32C 之外的 `Checksum` 实现时。

```java
var value = ScxDigest.digest(data, Adler32::new);
```
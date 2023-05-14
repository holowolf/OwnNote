## 2FA双因素认证

### 1. 前言

双重认证(英语：Two-factor authentication,缩写为2FA), 又译为双重验证、双因子认证、双因素认证、二元认证,又称两步骤验证(2-Step Verification,又译两步验证), 是一种认证方法,使用两种不同的元素,合并在一起,来确认用户的身份,是多因素验证中的一个特例.

* 使用银行卡时,需要另外输入PIN码,确认之后才能使用其转账功能.
* 登陆电脑版微信时,用已经登录同一账号的手机版微信扫描特定二维码进行验证.
* 登陆校园网系统时,通过手机短信或学校指定的手机软件进行验证.
* 登陆Steam和Uplay等游戏平台时,使用手机令牌或Google身份验证器进行验证.

### 2. TOTP

TOTP 的全称是”基于时间的一次性密码”(Time-based One-time Password). 它是公认的可靠解决方案,已经写入国际标准 RFC6238.

* 第一步,用户开启双因素认证后,服务器生成一个密钥.
* 第二步：服务器提示用户扫描二维码(或者使用其他方式),把密钥保存到用户的手机.也就是说,服务器和用户的手机,现在都有了同一把密钥.
* 第三步,用户登录时,手机客户端使用这个密钥和当前时间戳,生成一个哈希,有效期默认为30秒.用户在有效期内,把这个哈希提交给服务器.(注意,密钥必须跟手机绑定.一旦用户更换手机,就必须生成全新的密钥.)
* 第四步,服务器也使用密钥和当前时间戳,生成一个哈希,跟用户提交的哈希比对.只要两者不一致,就拒绝登录.

### 3.RFC6238

根据RFC 6238标准,供参考的实现如下：

* 生成一个任意字节的字符串密钥K,与客户端安全地共享.
* 基于T0的协商后,Unix时间从时间间隔(TI)开始计数时间步骤,TI则用于计算计数器C(默认情况下TI的数值是T0和30秒)的数值
* 协商加密哈希算法(默认为SHA-1)
* 协商密码长度(默认6位)

### 4. 2FA双因素认证 Golang 代码实现 TOTP

生成一次性密码的伪代码

```go
function GoogleAuthenticatorCode(string secret)
  key := base32decode(secret)
  message := floor(current Unix time / 30)
  hash := HMAC-SHA1(key, message)
  offset := last nibble of hash
  truncatedHash := hash[offset..offset+3]  //4 bytes starting at the offset
  Set the first bit of truncatedHash to zero  //remove the most significant bit
  code := truncatedHash mod 1000000
  pad code with 0 until length of code is 6
  return code
```

生成事件性或计数性的一次性密码伪代码

```go
function GoogleAuthenticatorCode(string secret)
  key := base32decode(secret)
  message := counter encoded on 8 bytes
  hash := HMAC-SHA1(key, message)
  offset := last nibble of hash
  truncatedHash := hash[offset..offset+3]  //4 bytes starting at the offset
  Set the first bit of truncatedHash to zero  //remove the most significant bit
  code := truncatedHash mod 1000000
  pad code with 0 until length of code is 6
  return code
```

* 关于代码中为什么会出现难懂的位运算 -> 追求运算效率

```go
package tf_authentication

import (
	"crypto/hmac"
	"crypto/sha1"
	"encoding/binary"
	"github.com/sirupsen/logrus"
	"time"
)

func UseHotp()  {
	key := []byte("MOJOTV_CN_IS_AWESOME_AND_AWESOME_SECRET_KEY")
	number := totp(key, time.Now(), 6)
	logrus.Info("2FA code: ", number)
}

func hotp(key []byte, counter uint64, digits int) int {
	// RFC 6238
	hash := hmac.New(sha1.New, key)
	_ = binary.Write(hash, binary.BigEndian, counter)
	sum := hash.Sum(nil)
	// 取sha1的最后4byte
	// 0x7FFFFFFF 是 long int的最大值
	// math.MaxUint32 == 2^32-1
	// & 0x7FFFFFFF == 2^31
	// len(sum)-1 ]&0x0F
	// 取sha1 bytes最后4byte 转换成 uint32
	u := binary.BigEndian.Uint32(sum[sum[len(sum)-1]&0x0F:]) & 0x7FFFFFFF
	d := uint32(1)

	// 取十进制余数
	for i := 0; i < digits && i < 8; i++ {
		d *= 10
	}
	return int(u % d)
}

func totp(key []byte, t time.Time, digits int) int {
	return hotp(key, uint64(t.Unix())/30, digits)
}
```

运行结果
```
time="2020-07-13T16:05:25+08:00" level=info msg="2FA code: 775300"
```

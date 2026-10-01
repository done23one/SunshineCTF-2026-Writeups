# SunshineCTF 2026 · kidding：JWT 的 kid 被当成文件路径

Web｜`kidding.web.2026.sunshinectf.games`｜`sun{h0tw1r3d_4dm1n_jwt}`

题名是 kid + ding。

拿到题先看首页，站点用 JWT 做会话，header 里的 `kid` 值长成一个文件名。我第一反应是：这个字段会不会被拿去拼路径？如果真是，那它读到什么文件，这把签名密钥就是什么。

顺着这条线往下走，最后发现编辑器密钥被硬编码在源码里，而源码正好能从同一个口子读到。

## 领证

```http
POST /login HTTP/1.1
Host: kidding.web.2026.sunshinectf.games
Content-Type: application/x-www-form-urlencoded
Content-Length: 11

name=reader
```

302，token 在中间响应的 `Set-Cookie` 上。在 Burp 里拦下来就能看到——要是让请求自动跟随重定向，这行头会跟着中间响应一起丢掉，我一开始就以为登录失败了。token 长这样：

```
eyJhbGciOiJIUzI1NiIsImtpZCI6InJlYWRlci5rZXkiLCJ0eXAiOiJKV1QifQ.eyJzdWIiOiJyZWFkZXIiLCJyb2xlIjoicmVhZGVyIn0._8O4qsSOc84vlXESt0W2VNygcuzLAZm0roqDZZKcU8A
```

三段，中间用点隔开。前两段是 base64url 编码不是加密，解码出来就是明文 JSON：

```
{"alg":"HS256","kid":"reader.key","typ":"JWT"}
{"sub":"reader","role":"reader"}
```

`kid` 的值是 `reader.key`，越看越像文件名。

## 硬改身份不行

先按常规思路试。带原证打 `/admin` 是 403，说明证是真的，只是身份不够。

那就直接改 payload，把 `role` 换成 `editor`，签名保持不动——结果是 401。

401 而不是 403，这个差别让我换了方向。403 是"你是 reader，这里要 editor"，401 是"你这张证本身是假的"。服务端先验章再看 role，说明我改的 `role` 根本没被读到。

改 payload 这条路封死了，只能自己签一张。而签名用哪把钥匙是由 `kid` 决定的，于是问题变成：`kid` 指向的钥匙，我能不能拿到？

## kid 是文件路径

先试 `kid` 到底被怎么用。我给它塞了个不存在的名字：

```
kid = nope.key
↓
KEY LOAD FAILED :: [Errno 2] No such file or directory:
  '/app/keys/nope.key'
```

报错把完整路径吐了出来。这说明它不是拿 `kid` 去查表，而是真的拼出一个路径去开文件，拼法是 `/app/keys/` 加 `kid`。

![kid=nope.key 时服务端的报错页面：它把 /app/keys/nope.key 这个内部路径完整打了出来](截图/09-报错泄漏路径.png)

如果 `kid` 是路径，那读哪个文件就由我说了算。

## 能读到什么

顺着这个口子探边界：

![探测三个文件的返回：系统文件的内容被原样打印，flag 和密钥文件都是 withheld](截图/09-探测结果.png)

第一行的内容被原样打印，任意文件读成立。但另外两行——flag 和那个密钥文件——都被挡了。

注意返回的文案是 `withheld`，不是"打开失败"。这个用词差别让我起疑：挡住的是打印，还是读取本身？要弄清这点，得看代码。

## 读源码，一次看清三件事

把 `kid` 指向程序自己：`kid=/app/app.py`。

第一件是 `kid` 怎么变成路径：

```python
def key_path(kid):
    return os.path.join(KEYS_DIR, kid)      # KEYS_DIR = /app/keys

def load_key(kid):
    with open(key_path(kid), 'rb') as f:
        return f.read()
```

`os.path.join` 的第二个参数以 `/` 开头时，前面那段会被整个丢掉。所以 `kid` 填一个以 `/` 开头的绝对路径，拼出来就是那个路径本身——一个斜杠就够，连 `..` 都不需要。

第二件是那个 `withheld` 是怎么来的：

```python
except Exception as e:
    if is_protected(kid):
        withheld = True         # ← 只影响"要不要打印"
    else:
        disclosed = key.decode('utf-8', errors='replace')   # ← 才展示
```

`load_key()` 早就执行完，`is_protected()` 只决定要不要打印。所谓"读不到"是假象——文件内容是读到了，只是不给你看。黑名单是一张固定的路径表，换个路径就能绕开。

第三件，也是最关键的，编辑器密钥本身：

```python
READER_KEY = secrets.token_hex(32).encode()   # 每次启动随机
EDITOR_KEY = b'ch-pr0t0type-edit0r-s1gn1ng-k3y-d0-n0t-sh1p'

with open(os.path.join(KEYS_DIR, 'editor.key'), 'wb') as f:
    f.write(EDITOR_KEY)      # 同一把钥匙再写进那个文件
```

同一把钥匙存了两处。`/app/keys/editor.key` 是数据文件，在黑名单里，读不到；源码里那一行是代码，不在黑名单里，读得到。

也就是说，`editor.key` 这个文件的内容，就等于源码里的 `EDITOR_KEY`。文件被挡了，但造出这个文件的代码没被挡——保护了数据，忘了保护生成数据的代码。

提取的时候别手抄，直接从返回的源码里把那段常量取出来用。抄错一个字符签名就全废，而这种错误看起来完全是对的。

## 伪造证

JWT 的第三段是 HMAC-SHA256 对前两段算出来的签名。密钥到手，把 payload 的 `role` 改成 `editor`，用 `EDITOR_KEY` 重新签一遍，就得到一张编辑证：

```
{"alg":"HS256","kid":"editor.key","typ":"JWT"}
{"sub":"reader","role":"editor"}
```

把这两段和 `EDITOR_KEY` 填进工具，算法选 HS256 就能签出来。

```http
GET /admin
Cookie: token=<伪造的证>
```

flag 在页面上。

![用 EDITOR_KEY 伪造的证访问 /admin：HTTP 200，Editor Access Granted，flag 直接印在页面上](截图/09-伪造证通关页面.png)

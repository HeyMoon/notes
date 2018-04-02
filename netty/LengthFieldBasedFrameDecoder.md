LengthFieldBasedFrameDecoder是一个decoder,它将接收的`ByteBuf`根据message中的长度字段动态的切开。当你要解码一个二进制消息，它有一个整数头部字段代表消息体或者是整个消息的长度时，这个解码器特别有用。

如上述，你可能会碰到这样一下协议，他们的格式如下：

````
+--------+----------------+
| Length |   message body |
+--------+----------------+
````

其中，这个`Length`代表`message body`的长度或整个消息的长度。

这个时候你就可以使用`LengthFieldBasedFrameDecoder`作为解码器。

我们来看一下`LengthFieldBasedFrameDecoder`的构造方法：

````
public LengthFieldBasedFrameDecoder(
            int maxFrameLength,
            int lengthFieldOffset, int lengthFieldLength,
            int lengthAdjustment, int initialBytesToStrip)
````

其中：

+ `maxFrameLength`表示这个帧的最大长度，如果帧的长度超过了`maxFrameLength`，会抛出`TooLongFrameException`。
+ `lengthFieldOffset`表示长度字段在帧中的偏移。
+ `lengthFieldLength`表示长度字段占的字节数（只能为1，2，3，4，8）；
+ `lengthAdjustment`表示长度调整值。
+ `initialBytesToStrip`表示从第一个字节开始舍弃的字节数。

很抽象是不是？我们用几个例子来详细解释每个字段的含义。

其实最关键的还是长度字段的含义以及是否舍弃字节（initialBytesToStrip）：长度字段是表示消息体的长度，还是整个消息的长度。

长度字段表示的frameLength可以通过下列代码算出：

````
1:frameLength = buf.getUnsignedByte(offset);
2:frameLength = buf.getUnsignedShort(offset);
3:frameLength = buf.getUnsignedMedium(offset);
4:frameLength = buf.getUnsignedInt(offset);
8:frameLength = buf.getLong(offset);
````

那么，在这里先给一个计算解码后消息的长度的公式：

afterDecodelLength = `buffer.get(offset)` + `lengthAdjustment` + `lengthFieldOffset` + `lengthFieldLength` - `initialBytesToStrip`

这里的`buffer.get(offset)`对应上面的`1，2，3，4，8`。

### 例1
协议为：长度字段offset为0，占2个字节（byte），长度字段代表消息体的长度，不舍弃字节,如下所示：

````
  BEFORE DECODE (14 bytes)         AFTER DECODE (14 bytes)
 +--------+----------------+      +--------+----------------+
 | Length | Actual Content |----->| Length | Actual Content |
 | 0x000C | "HELLO, WORLD" |      | 0x000C | "HELLO, WORLD" |
 +--------+----------------+      +--------+----------------+
````

那么：

+ 因为offset是0，所以`lengthFieldOffset = 0`
+ 因为长度字段占2字节，所以`lengthFieldLength = 2`
+ 因为不舍弃字节，所以`initialBytesToStrip = 0`
+ 按照协议，解码之后的长度和解码之前的长度一致。因为没有舍弃字节而且长度字段表示消息体的长度，所以`lengthAdjustment = 0`。也可以由上面的afterDecodelLength的计算公式可以算出`lengthAdjustment = 0`

### 例2
协议为：长度字段offset为0，占2个字节（byte），长度字段代表消息体的长度，舍弃头部2字节,如下所示：

````
  BEFORE DECODE (14 bytes)         AFTER DECODE (12 bytes)
  +--------+----------------+      +----------------+
  | Length | Actual Content |----->| Actual Content |
  | 0x000C | "HELLO, WORLD" |      | "HELLO, WORLD" |
  +--------+----------------+      +----------------+
````

那么：

+ 因为offset是0，所以`lengthFieldOffset = 0`
+ 因为长度字段占2字节，所以`lengthFieldLength = 2`
+ 因为舍弃头部2字节,所以`initialBytesToStrip = 2`
+ 按照协议，要求舍弃头部2字节，所以，`lengthAdjustment = 0`。也可以根据上面的afterDecodelLength计算，`lengthAdjustment` = 0

### 例3
协议为：长度字段表示整个消息的长度，长度字段offset为0，占2个字节，不舍弃字节。

````
 BEFORE DECODE (14 bytes)         AFTER DECODE (14 bytes)
 +--------+----------------+      +--------+----------------+
 | Length | Actual Content |----->| Length | Actual Content |
 | 0x000E | "HELLO, WORLD" |      | 0x000E | "HELLO, WORLD" |
 +--------+----------------+      +--------+----------------+
````

因为协议要求解码后不舍弃字节，所以：

+ `lengthFieldOffset = 0`
+ `lengthFieldLength = 2`
+ `initialBytesToStrip = 0`
+ 根据afterDecodelLength公式算出：`lengthAdjustment = -2`

### 例4
协议为：头部为5个字节，length字段在头部的后3个字节，length字段表示消息体的长度，不舍弃头部。如下所示：

````
 BEFORE DECODE (17 bytes)                      AFTER DECODE (17 bytes)
 +----------+----------+----------------+      +----------+----------+----------------+
 | Header 1 |  Length  | Actual Content |----->| Header 1 |  Length  | Actual Content |
 |  0xCAFE  | 0x00000C | "HELLO, WORLD" |      |  0xCAFE  | 0x00000C | "HELLO, WORLD" |
 +----------+----------+----------------+      +----------+----------+----------------+
````

所以：

+ `lengthFieldOffset = 2`
+ `lengthFieldLength = 3`
+ `initialBytesToStrip = 0`
+ 根据afterDecodelLength公式算出：`lengthAdjustment = 0`

### 例5
协议为：头部为5字节，length字段在头部的前3字节，length字段表示消息体长度，不舍弃头部。如下所示：

````
  BEFORE DECODE (17 bytes)                      AFTER DECODE (17 bytes)
  +----------+----------+----------------+      +----------+----------+----------------+
  |  Length  | Header 1 | Actual Content |----->|  Length  | Header 1 | Actual Content |
  | 0x00000C |  0xCAFE  | "HELLO, WORLD" |      | 0x00000C |  0xCAFE  | "HELLO, WORLD" |
  +----------+----------+----------------+      +----------+----------+----------------+
````

所以:

+ `lengthFieldOffset = 0`
+ `lengthFieldLength = 3`
+ `initialBytesToStrip = 0`
+ 因为length字段表示消息体长度，根据afterDecodelLength公式算出: `lengthAdjustment = 2`

### 例6
协议为：头部为4字节，length字段在offset为1之后的2字节，length字段表示消息体长度，舍弃第一个头部字段（就是length字段之前的一个字节）和length字段。如下所示：

````
 BEFORE DECODE (16 bytes)                       AFTER DECODE (13 bytes)
 +------+--------+------+----------------+      +------+----------------+
 | HDR1 | Length | HDR2 | Actual Content |----->| HDR2 | Actual Content |
 | 0xCA | 0x000C | 0xFE | "HELLO, WORLD" |      | 0xFE | "HELLO, WORLD" |
 +------+--------+------+----------------+      +------+----------------+
````

所以：

+ `lengthFieldOffset = 1`
+ `lengthFieldLength = 2`
+ `initialBytesToStrip = 3`(第一个头部字段 + length字段 的长度)
+ 因为length字段表示消息体长度，根据afterDecodelLength公式算出：`lengthAdjustment = 1`

### 例7
协议为：头部为4字节，length字段在offset为1之后的2字节，length字段表示整个消息长度，舍弃第一个头部字段（就是length字段之前的一个字节）和length字段。如下所示：

````
 BEFORE DECODE (16 bytes)                       AFTER DECODE (13 bytes)
 +------+--------+------+----------------+      +------+----------------+
 | HDR1 | Length | HDR2 | Actual Content |----->| HDR2 | Actual Content |
 | 0xCA | 0x0010 | 0xFE | "HELLO, WORLD" |      | 0xFE | "HELLO, WORLD" |
 +------+--------+------+----------------+      +------+----------------+
````

所以：

+ `lengthFieldOffset = 1`
+ `lengthFieldLength = 2`
+ `initialBytesToStrip = 3`
+ 因为length表示整个消息长度，根据afterDecodelLength算出：`lengthAdjustment = -3`

## 总结
最重要的协议。协议规定解码后要不要舍弃头部，长度字段表示什么（消息体长度还是整个消息长度）。这些确定好了，`lengthAdjustment`就可以确定了。

[例子来源:](http://netty.io/4.0/api/io/netty/handler/codec/LengthFieldBasedFrameDecoder.html)

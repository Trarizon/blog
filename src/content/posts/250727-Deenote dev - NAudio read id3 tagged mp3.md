---
title: Deenote dev - NAudio读取带封面mp3
published: 2025-07-27
description: 关于修复NAudio对mp3文件ID3 tag的处理.
tags: [Deenote, C#, NAudio, mp3]
category: Deenote
draft: false
---

Deenote 1.0.4终于修复了鸽了十万年的带封面mp3导入的问题。由于笔者本地使用foobar2000管理Deemo收录曲时添加了自定义tag，导致dnt无法直接读取，于是在今天测试的时候顺手查了下如何修复。

## NAudio的问题

NAudio对mp3的ID3v2的支持似乎不完整，导致有时候读取带ID3v2的文件时出错，而mp3的封面也是ID3 tag的一部分。

那手搓

## Mp3文件的ID3v2 tag

Mp3文件头部带有的ID3头数据格式如下
``` csharp
struct ID3Header
{
    fixed byte Identifier[3]; // 标识符，值为"ID3"u8
    byte Version;             // ID3版本号，ID3V2.3值为3
    byte Revision;            // 副版本号
    byte Flags;
    fixed byte Size[4];       // tag大小
}
```

其中tag大小为Big Endian的四字节，每个字节的存储7 bit值，首位不使用，为0。
比如当size为0b00001100_11110011时，tag的值应当为0b00011001 0b01110011



## Deenote内的具体实现

我们的目的是只是跳过ID3tag的读取，所以只需要`Identifer`和`Size`的值。读取`ID3Header`的数据，并跳过后续`Size`大小的数据

``` csharp
void SkipID3Tag(Stream stream, out int id3TagLength)
{
    var resetPosition = stream.Position;

    Span<byte> identifier = stackalloc byte[3];
    stream.Read(identifier);
    if (id3 is "ID3"u8)
    {
        stream.Position += 3; // 跳过不需要的字节
        Span<byte> sizeData = stackalloc byte[4];
        stream.Read(sizeData);
        
        uint size = 0;
        // Big-Edian读取，size首位一定为0，因此不需要特地& 0x7F，每个字节只有7bits值，因此左移7的倍数
        size |= (uint)sizeData[0] << 21;
        size |= (uint)sizeData[1] << 14;
        size |= (uint)sizeData[2] << 7;
        size |= sizeData[3];

        stream.Position += size;
        id3TagLength = 10 + (checked)((int)size);
    }
    else
    {
        // 没有tag，直接返回
        stream.Position = resetPosition;
        id3TagLength = 0;
    }
}
```

> References
> - [ID3V2_header](https://id3.org/id3v2.3.0#ID3v2_header)
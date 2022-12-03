---
title: java版3des加密程序，可与php兼容
author: 阿辉
date: 2008-12-04T16:45:00+00:00
categories:
- Php
tags:
- Php
keywords:
- Php
comments: true
showTags: true
showPagination: true
showSocial: false
showDate: true
showMeta: true
---

写了一个java版3des加密程序，可与php兼容，代码如下：

<!--more-->

```java
import java.io.UnsupportedEncodingException;
import java.security.Key;
import javax.crypto.Cipher;
import javax.crypto.SecretKeyFactory;
import javax.crypto.spec.DESedeKeySpec;
import javax.crypto.spec.IvParameterSpec;
import javax.crypto.spec.SecretKeySpec;
import sun.misc.BASE64Decoder;
import sun.misc.BASE64Encoder;

public class DESCoder
{
    private static BASE64Encoder base64 = new BASE64Encoder();
    private static byte[] myIV = { 50, 51, 52, 53, 54, 55, 56, 57 };
    //private static String strkey = “W9qPIzjaVGKUp7CKRk/qpCkg/SCMkQRu”; // 字节数必须是8的倍数
    private static String strkey = “01234567890123456789012345678912”;
    public static String desEncrypt(String input) throws Exception
    {
       
        BASE64Decoder base64d = new BASE64Decoder();
        DESedeKeySpec p8ksp = null;
        p8ksp = new DESedeKeySpec(base64d.decodeBuffer(strkey));
        Key key = null;
        key = SecretKeyFactory.getInstance(“DESede”).generateSecret(p8ksp);
       
        input = padding(input);
        byte[] plainBytes = (byte[])null;
        Cipher cipher = null;
        byte[] cipherText = (byte[])null;
       
        plainBytes = input.getBytes(“UTF8”);
        cipher = Cipher.getInstance(“DESede/CBC/NoPadding”);
        SecretKeySpec myKey = new SecretKeySpec(key.getEncoded(), “DESede”);
        IvParameterSpec ivspec = new IvParameterSpec(myIV);
        cipher.init(1, myKey, ivspec);
        cipherText = cipher.doFinal(plainBytes);
        return removeBR(base64.encode(cipherText));
       
    }
    
    public static String desDecrypt(String cipherText) throws Exception
    {
       
        BASE64Decoder base64d = new BASE64Decoder();
        DESedeKeySpec p8ksp = null;
        p8ksp = new DESedeKeySpec(base64d.decodeBuffer(strkey));
        Key key = null;
        key = SecretKeyFactory.getInstance(“DESede”).generateSecret(p8ksp);
       
        Cipher cipher = null;
        byte[] inPut = base64d.decodeBuffer(cipherText);
        cipher = Cipher.getInstance(“DESede/CBC/NoPadding”);
        SecretKeySpec myKey = new SecretKeySpec(key.getEncoded(), “DESede”);
        IvParameterSpec ivspec = new IvParameterSpec(myIV);
        cipher.init(2, myKey, ivspec);
        byte[] output = removePadding(cipher.doFinal(inPut));

        return new String(output, “UTF8”);
       
    }
    
    private static String removeBR(String str) {
        StringBuffer sf = new StringBuffer(str);

        for (int i = 0; i < sf.length(); ++i)
        {
          if (sf.charAt(i) ‘n’)
          {
            sf = sf.deleteCharAt(i);
          }
        }
        for (int i = 0; i < sf.length(); ++i)
          if (sf.charAt(i) ‘r’)
          {
            sf = sf.deleteCharAt(i);
          }

        return sf.toString();
      }

      public static String padding(String str)
      {
        byte[] oldByteArray;
        try
        {
            oldByteArray = str.getBytes(“UTF8”);
            int numberToPad = 8 - oldByteArray.length % 8;
            byte[] newByteArray = new byte[oldByteArray.length + numberToPad];
            System.arraycopy(oldByteArray, 0, newByteArray, 0, oldByteArray.length);
            for (int i = oldByteArray.length; i < newByteArray.length; ++i)
            {
                newByteArray[i] = 0;
            }
            return new String(newByteArray, “UTF8”);
        }
        catch (UnsupportedEncodingException e)
        {
            System.out.println(“Crypter.padding UnsupportedEncodingException”);
        }
        return null;
      }
      public static byte[] removePadding(byte[] oldByteArray)
      {
        int numberPaded = 0;
        for (int i = oldByteArray.length; i >= 0; –i)
        {
          if (oldByteArray[(i - 1)] != 0)
          {
            numberPaded = oldByteArray.length - i;
            break;
          }
        }

        byte[] newByteArray = new byte[oldByteArray.length - numberPaded];
        System.arraycopy(oldByteArray, 0, newByteArray, 0, newByteArray.length);

        return newByteArray;
      }
     
    public static void main(String args[])
    {
        try {
            String desstr = DESCoder.desEncrypt(“1qaz2ws”);
            String pstr = DESCoder.desDecrypt(desstr);
            System.out.println(“plainText:1qaz2ws”);
            System.out.println(“Encode:”+desstr);
            System.out.println(“Decode:”+pstr);
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```

运行结果：
```
plainText:1qaz2ws
Encode:0GsXgYA8BuM=
Decode:1qaz2ws
```

和PHP的一样，呵呵。
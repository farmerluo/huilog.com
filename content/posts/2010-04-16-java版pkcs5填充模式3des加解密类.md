---
title: JAVA版PKCS5填充模式3DES加解密类
author: 阿辉
date: 2010-04-16T09:59:00+00:00
categories:
- JAVA
tags:
- JAVA
keywords:
- JAVA
comments: true
showTags: true
showPagination: true
showSocial: false
showDate: true
showMeta: true
---
DESCoder.java:
```java
/*
* To change this template, choose Tools | Templates
* and open the template in the editor.
*/

package des;
import java.security.Key;
import javax.crypto.Cipher;
import javax.crypto.SecretKeyFactory;
import javax.crypto.spec.DESedeKeySpec;
import javax.crypto.spec.IvParameterSpec;
import javax.crypto.spec.SecretKeySpec;
import sun.misc.BASE64Decoder;
import sun.misc.BASE64Encoder;

/**
*
* @author FarmerLuo
*/
public class DESCoder {
    private static BASE64Encoder base64 = new BASE64Encoder();
    private static byte[] myIV = { 50, 51, 52, 53, 54, 55, 56, 57 };
    //private static String strkey = "W9qPIzjaVGKUp7CKRk/qpCkg/SCMkQRu"; // 字节数必须是8的倍数
    private static String strkey = "01234567890123456789012345678912";
    public static String desEncrypt(String input) throws Exception
    {

        BASE64Decoder base64d = new BASE64Decoder();
        DESedeKeySpec p8ksp = null;
        p8ksp = new DESedeKeySpec(base64d.decodeBuffer(strkey));
        Key key = null;
        key = SecretKeyFactory.getInstance("DESede").generateSecret(p8ksp);

        byte[] plainBytes = (byte[])null;
        Cipher cipher = null;
        byte[] cipherText = (byte[])null;

        plainBytes = input.getBytes("UTF8");
        cipher = Cipher.getInstance("DESede/CBC/PKCS5Padding");
        SecretKeySpec myKey = new SecretKeySpec(key.getEncoded(), "DESede");
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
        key = SecretKeyFactory.getInstance("DESede").generateSecret(p8ksp);

        Cipher cipher = null;
        byte[] inPut = base64d.decodeBuffer(cipherText);
        cipher = Cipher.getInstance("DESede/CBC/PKCS5Padding");
        SecretKeySpec myKey = new SecretKeySpec(key.getEncoded(), "DESede");
        IvParameterSpec ivspec = new IvParameterSpec(myIV);
        cipher.init(2, myKey, ivspec);
        byte[] output = cipher.doFinal(inPut);
        return new String(output, "UTF8");

    }

    private static String removeBR(String str) {
        StringBuffer sf = new StringBuffer(str);

        for (int i = 0; i < sf.length(); ++i)
        {
          if (sf.charAt(i) 'n')
          {
            sf = sf.deleteCharAt(i);
          }
        }
        for (int i = 0; i < sf.length(); ++i)
          if (sf.charAt(i) 'r')
          {
            sf = sf.deleteCharAt(i);
          }

        return sf.toString();
      }
}
```
<!--more-->
Main.java:
```java
package des;

/**
*
* @author FarmerLuo
*/
public class Main {

    /**
     * @param args the command line arguments
     */
    public static void main(String[] args) {
        try {
            String desstr = DESCoder.desEncrypt("12354");
            String pstr = DESCoder.desDecrypt(desstr);
            System.out.println("plainText:12354");
            System.out.println("Encode:"+desstr);
            System.out.println("Decode:"+pstr);
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

}
```
运行结果：
```
run：
plainText:12354
Encode:pzilYggKgBU=
Decode:12354
```
成功生成（总时间：1 秒）
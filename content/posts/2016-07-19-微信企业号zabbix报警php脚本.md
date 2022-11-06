---
title: 微信企业号Zabbix报警PHP脚本
author: 阿辉
date: 2016-07-19T08:43:39+00:00
categories:
- Zabbix
tags:
- Zabbix
- Wechat
keywords:
- Zabbix
- Wechat
comments: true
showTags: true
showPagination: true
showSocial: false
showDate: true
showMeta: true
clearReading: false
---
配置ZABBIX的微信报警时，发现网上的python有兼容性问题，于是自己用PHP写了一个发送的脚本，放在这里给需要的人。

<!--more-->

```php
#!/usr/bin/php
<?php

$corp_id    = 'xxxxx';               //微信企业号ID
$secret     = 'jS8WzmdueRIZKxxxE';   //微信企业号应用secret
$app_id     = 1;                     //微信企业号应用ID号

// display E_ERROR and E_WARNING and E_PARSE
error_reporting(7);
date_default_timezone_set('Asia/Chongqing');

if ( $argc < 3 ) {
   echo "usage:php ${argv[0]} weChat_userId subject content\n";
   exit;
}

//var_dump($argv);
$userId  = $argv[1];
$subject = $argv[2];
$content = $argv[3];

$wx = new weChat($corp_id, $secret, $app_id);
$wx->sendText($userId, $subject, $content);


class weChat{
    private $corpId;
    private $corpSecret;
    private $appId;
    private $tokenFile  = "/tmp/wx_token.txt";
    public $token;

    function __construct($corpId, $corpSecret, $app_id)
    {
        openlog('SendWeChat', LOG_PID | LOG_ODELAY,LOG_LOCAL4);
        $this->corpId = $corpId;
        $this->corpSecret = $corpSecret;
        $this->appId = $app_id;
    }

    function __destruct()
    {
        closelog();
    }

    /**
     * 得到微信企业号Token
     * @return string token
     */
    public function getToken()
    {
        // 微信Token API URL
        $url = "https://qyapi.weixin.qq.com/cgi-bin/gettoken?corpid={$this->corpId}&corpsecret={$this->corpSecret}";

        if ( $this->getSaveToken() ) {
            return true;
        } else {
            $re  = $this->http_get($url);
            if ($re === false) {
                syslog(LOG_INFO, "get weixin token false.");
                return false;
            }

            $token = json_decode($re,true);
            if( array_key_exists("access_token", $token) ) {
                $token['expire_time'] = $token['expires_in'] + time();
                $this->saveToken(json_encode($token));
                $this->token = $token;
                return true;
            } else {
                syslog(LOG_INFO, "get weixin token error,access_token not exists.");
                return false;
            }
        }

    }

    /**
     * 发送文本消息
     * @param  string $userId
     * @param  string $subject
     * @param  string $content
     * @return bool
     */
    public function sendText($userId, $subject, $content)
    {
        // 微信Token API POST URL
        $url = "https://qyapi.weixin.qq.com/cgi-bin/message/send?access_token=";

        if ( $this->getToken() ) {
            $data = array(
                'touser'  => $userId,
                'msgtype' => 'text',
                'agentid' => $this->appId,
                'text'    => array('content' => urlencode($subject . "\n\n" . $content))
            );
            $re = $this->http_post($url . $this->token['access_token'], urldecode(json_encode($data)));
            if ($re === false) {
                syslog(LOG_INFO, "send text messages to weixin({$userId}) false.");
                return false;
            }

            $arrayRe = json_decode($re,true);
            if( $arrayRe['errcode'] == 0 ) {
                syslog(LOG_INFO, "send text messages to weixin({$userId}) succeed.");
                return true;
            } else {
                syslog(LOG_INFO, "send text messages to weixin({$userId}) false! errcode:{$arrayRe['errcode']} errmsg:{$arrayRe['errmsg']} ");
                return false;
            }
        } else {
            return false;
        }

    }

    /**
     * 保存Token到临时文件
     * @param string $token
     * @return bool
     *
     */
    private function saveToken( $token ){
        try {
            file_put_contents($this->tokenFile, $token, LOCK_EX);
            return true;
        } catch (Exception $e) {
            syslog(LOG_INFO, "Write File Error (File: ".$e->getFile().", line ".
                $e->getLine()."): ".$e->getMessage());
            return false;
        }

    }

    /**
     * 得到保存的Token
     * @return  bool
     */
    private function getSaveToken(){
        if (file_exists($this->tokenFile) && is_readable($this->tokenFile)) {
            $token = json_decode(file_get_contents($this->tokenFile),true);
            $now = time() + 60;
            if ( $token['expire_time'] > $now ) {
                $this->token = $token;
                return true;
            } else
                return false;
        } else {
            syslog(LOG_INFO, "file({$this->tokenFile}) not exists on unreadable.");
            return false;
        }
    }

    /**
     * GET 请求
     * @param string $url
     * @return string content
     */
    public function http_get($url){
        $oCurl = curl_init();
        if(stripos($url,"https://")!==FALSE){
            curl_setopt($oCurl, CURLOPT_SSL_VERIFYPEER, FALSE);
            curl_setopt($oCurl, CURLOPT_SSL_VERIFYHOST, FALSE);
            curl_setopt($oCurl, CURLOPT_SSLVERSION, 1); //CURL_SSLVERSION_TLSv1
        }
        curl_setopt($oCurl, CURLOPT_URL, $url);
        curl_setopt($oCurl, CURLOPT_RETURNTRANSFER, 1 );
        $sContent = curl_exec($oCurl);
        $aStatus = curl_getinfo($oCurl);
        curl_close($oCurl);
        if(intval($aStatus["http_code"])==200){
            return $sContent;
        }else{
            return false;
        }
    }

    /**
     * POST 请求
     * @param string $url
     * @param array $param
     * @param boolean $post_file 是否文件上传
     * @return string content
     */
    public function http_post($url,$param,$post_file=false){
        $oCurl = curl_init();
        if(stripos($url,"https://")!==FALSE){
            curl_setopt($oCurl, CURLOPT_SSL_VERIFYPEER, FALSE);
            curl_setopt($oCurl, CURLOPT_SSL_VERIFYHOST, false);
            curl_setopt($oCurl, CURLOPT_SSLVERSION, 1); //CURL_SSLVERSION_TLSv1
        }
        if (is_string($param) || $post_file) {
            $strPOST = $param;
        } else {
            $aPOST = array();
            foreach($param as $key=>$val){
                $aPOST[] = $key."=".urlencode($val);
            }
            $strPOST =  join("&", $aPOST);
        }
        //var_dump($strPOST);
        //$header = array('Content-type: application/json');
        //curl_setopt($oCurl, CURLOPT_HTTPHEADER, $header);
        curl_setopt($oCurl, CURLOPT_URL, $url);
        curl_setopt($oCurl, CURLOPT_RETURNTRANSFER, 1 );
        curl_setopt($oCurl, CURLOPT_POST,true);
        curl_setopt($oCurl, CURLOPT_POSTFIELDS,$strPOST);
        $sContent = curl_exec($oCurl);
        $aStatus = curl_getinfo($oCurl);
        curl_close($oCurl);
        if(intval($aStatus["http_code"])==200){
            return $sContent;
        }else{
            return false;
        }
    }
    
}
``` 
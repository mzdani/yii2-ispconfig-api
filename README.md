Yii2 Web Font Loader Wrapper
=============

[![Latest Stable Version](https://poser.pugx.org/mzdani/yii2-ispconfig-api/v/stable.svg)](https://packagist.org/packages/mzdani/yii2-ispconfig-api) [![Total Downloads](https://poser.pugx.org/mzdani/yii2-ispconfig-api/downloads.svg)](https://packagist.org/packages/mzdani/yii2-ispconfig-api) [![Latest Unstable Version](https://poser.pugx.org/mzdani/yii2-ispconfig-api/v/unstable.svg)](https://packagist.org/packages/mzdani/yii2-ispconfig-api) [![License](https://poser.pugx.org/mzdani/yii2-ispconfig-api/license)](https://packagist.org/packages/mzdani/yii2-ispconfig-api)

Installation
------------

The preferred way to install this extension is through [composer](http://getcomposer.org/download/).

Either run

```
php composer.phar require --prefer-dist "mzdani/yii2-ispconfig-api" "dev-master"
```

or add

```
"mzdani/yii2-ispconfig-api": "dev-master"
```

to the require section of your `composer.json` file.


Usage
-----

Below is some example how to use the wrapper.

```
#!php
<?php
use mdscomp\ISPConfig\SoapClient;

$cp = new SoapClient('http://your.cpdomain.tld:8080/remote/index.php', 'remoteusername', 'remoteuserpass');

// Get Quota Status from Client
$client_records = $cp->clientGetQuota('yourclientusername');

print_r($client_records);

?>
```

ISPConfig Custom Patch
-----

Because ISPConfig doesn't have any support to show quota usage with remote api. I must add new function into default ISPConfig remote api file. You can find it from `/usr/local/ispconfig/interface/lib/classes/remoting.inc.php`, open it and insert new function below into at the end of class.


```
#!php
<?php

	/*******************
	* CUSTOM PATCH START
	*******************/

	public function client_get_quota($session_id, $client_username){
		global $app;

		/*
		* Get Client Default Group ID
		*/
		$client_records = $this->client_get_by_username($session_id, $client_username);
		$client_data = $this->client_get($session_id, $client_records['client_id']);

		/*
		* Website Harddisk Quota
		*/
		$tmp_site =  $app->db->queryAllRecords("SELECT data from monitor_data WHERE type = 'harddisk_quota' ORDER BY created DESC");
		$monitor_data_site = array();
		if(is_array($tmp_site)) {
			foreach ($tmp_site as $tmp_mon_site) {
				$monitor_data_site = array_merge_recursive($monitor_data_site, unserialize($app->db->unquote($tmp_mon_site['data'])));
			}
		}

		$sites = $app->db->queryAllRecords("SELECT * FROM web_domain WHERE active = 'y' AND type = 'vhost' AND sys_groupid = ".$client_records['default_group']);
		$has_quota = false;
		$total_quota['limit_web_quota'] = ($client_data['limit_web_quota'] == '-1') ? $app->lng('unlimited') : $client_data['limit_web_quota']*1024;
		$total_quota['limit_traffic_quota'] = ($client_data['limit_traffic_quota'] == '-1') ? $app->lng('unlimited') : $client_data['limit_traffic_quota']*1024;
		$total_quota['limit_mailquota'] = ($client_data['limit_mailquota'] == '-1') ? $app->lng('unlimited') : $client_data['limit_mailquota']*1024;
		$web_quota = [];
		if(is_array($sites) && !empty($sites)){
			for($i=0;$i<sizeof($sites);$i++){
				$username = $sites[$i]['system_user'];
				$web_quota[$i]['domain'] = $sites[$i]['domain'];
				$web_quota[$i]['hd_quota'] = ($sites[$i]['hd_quota'] == '-1') ? $app->lng('unlimited') : $sites[$i]['hd_quota']*1024;
				$web_quota[$i]['traffic_quota'] = ($sites[$i]['traffic_quota'] == '-1') ? $app->lng('unlimited') : $sites[$i]['traffic_quota']*1024;
				$web_quota[$i]['used'] = $monitor_data_site['user'][$username]['used'];
				$web_quota[$i]['soft'] = $monitor_data_site['user'][$username]['soft'];
				$web_quota[$i]['hard'] = $monitor_data_site['user'][$username]['hard'];
				$web_quota[$i]['files'] = $monitor_data_site['user'][$username]['files'];

				if (!is_numeric($web_quota[$i]['used'])){
					if ($web_quota[$i]['used'][0] > $web_quota[$i]['used'][1]){
						$web_quota[$i]['used'] = $web_quota[$i]['used'][0];
					} else {
						$web_quota[$i]['used'] = $web_quota[$i]['used'][1];
					}
				}
				if (!is_numeric($web_quota[$i]['soft'])) $web_quota[$i]['soft']=$web_quota[$i]['soft'][1];
				if (!is_numeric($web_quota[$i]['hard'])) $web_quota[$i]['hard']=$web_quota[$i]['hard'][1];
				if (!is_numeric($web_quota[$i]['files'])) $web_quota[$i]['files']=$web_quota[$i]['files'][1];			

				if($web_quota[$i]['soft'] == 0) $web_quota[$i]['soft'] = $app->lng('unlimited');
				if($web_quota[$i]['hard'] == 0) $web_quota[$i]['hard'] = $app->lng('unlimited');

				if($web_quota[$i]['soft'] == '0 B' || $web_quota[$i]['soft'] == '0 KB' || $web_quota[$i]['soft'] == '0') $web_quota[$i]['soft'] = $app->lng('unlimited');
				if($web_quota[$i]['hard'] == '0 B' || $web_quota[$i]['hard'] == '0 KB' || $web_quota[$i]['hard'] == '0') $web_quota[$i]['hard'] = $app->lng('unlimited');
			}
			$has_quota = true;
		}

		/*
		* Website Harddisk Quota
		*/
		$tmp_mails =  $app->db->queryAllRecords("SELECT data from monitor_data WHERE type = 'email_quota' ORDER BY created DESC");
		$monitor_data_mails = array();
		if(is_array($tmp_mails)) {
			foreach ($tmp_mails as $tmp_mon_mails) {
				$tmp_array = unserialize($app->db->unquote($tmp_mon_mails['data']));
				if(is_array($tmp_array)) {
					foreach($tmp_array as $username => $data) {
						if(!$monitor_data_mails[$username]['used']) $monitor_data_mails[$username]['used'] = $data['used'];
					}
				}
			}
		}

		$emails = $app->db->queryAllRecords("SELECT * FROM mail_user WHERE 1 AND sys_groupid = ".$client_records['default_group']);
		$has_mailquota = false;
		$mails_quota = [];
		if(is_array($emails) && !empty($emails)){
			for($i=0;$i<sizeof($emails);$i++){
				$email = $emails[$i]['email'];

				$mails_quota[$i]['email'] = $emails[$i]['email'];
				$mails_quota[$i]['used'] = isset($monitor_data_mails[$email]['used']) ? $monitor_data_mails[$email]['used']*1024 : array(1 => 0);

				if (!is_numeric($mails_quota[$i]['used'])) $mails_quota[$i]['used']=$mails_quota[$i]['used'][1];

				if($mails_quota[$i]['quota'] == 0){
					$mails_quota[$i]['quota'] = $app->lng('unlimited');
				}
			}
			$has_mailquota = true;
		}


		/*
		* Returning Data
		*/
		$return = [
			'limit' => $total_quota,
			'used' => [
				'site' => ($has_quota) ? $web_quota : false,
				'mail' => ($has_mailquota) ? $mails_quota : false,
			],
		];
		return $return;
	}

	/*****************
	* CUSTOM PATCH END
	*****************/

?>
```
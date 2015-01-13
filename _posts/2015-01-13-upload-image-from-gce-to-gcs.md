---
layout: post
author: yukinasu
title: Google Compute Engine から Google Cloud StrageにImageをアップロードする
tags:
- GoogleComputeEngine
---

こんにちは、[yukinasu](https://twitter.com/yukinasu715)です。
今回は、Google Compute Engine(以降、GCEと記述)のImageをGoogle Cloud Strage(以降、GCSと記述)にアップロードする方法を紹介します。

この方法を利用すると
* 自分がカスタマイズしたImageを共有する
* 複数のサーバを立ち上げるときのBaseとなるCustomImageを作成する
ことが出来るようになります。

この記事はGCE公式ドキュメントの[Export an image to Google Cloud Storage](https://cloud.google.com/compute/docs/images#export_an_image_to_google_cloud_storage)を参考にしています。

## 1. temporary disk (一時保存用disk)を作成
このdiskはroot persistent disk (アップロード対象のdisk)をtarファイルとして固める際の、ファイル保存領域として使用される一時保存用のdiskとなります。
注意しなくてはいけないことは、このdiskの内容がtarに保存される訳ではないということです。

```bash
$ gcloud compute disks create TEMPORARY_DISK_NAME \
    --project PROJECT_NAME \
    --zone ZONE_NAME \
```

## 2. Instanceに一時保存用diskを適用
1.で作成したtemporary diskをInstanceに適用します。これには次の２通りの方法が有ります。

### 2.1 新規にInstanceを作成する場合
root persistent diskのInstanceがまだ作成されていない場合は、`gcloud compute instances create`コマンドを用いてインスタンスの作成と同時にtemporary diskの適用を行います。

```bash
$ gcloud compute instances create INSTANCE_NAME \
    --project PROJECT_NAME \
    --zone ZONE_NAME \
    --image IMAGE_NAME \
    --disk \
      name=TEMPORARY_DISK_NAME \
      device-name=TEMPORARY_DISK_NAME \
    --scopes storage-rw
```

GCSにアップロードするので、`--scopes storage-rw`オプションでストレージへの読み込み・書き込みの権限を付加します。

### 2.2 既存のInstanceに適用する場合
既に稼働しているInstanceに対してdiskを適用する場合は、`gcloud compute instances attach-disk`コマンドを用いてtemporary diskを適用します。

```bash
$ gcloud compute instances attach-disk INSTANCE_NAME \
    --project PROJECT_NAME \
    --zone ZONE_NAME \
    --disk TEMPORARY_DISK_NAME \
    --device-name TEMPORARY_DISK_NAME \
```

既存のInstanceに適用する場合は、Storageに対する書き込み権限がないとGCSにアップロードできないので、予め権限が存在することを確認しておいてください。

## 3. 起動したInstanceにSSHで接続
sshで接続するための設定に関してはGCE公式ドキュメントの[Connecting to an instance using ssh](https://cloud.google.com/compute/docs/instances#sshing)を参考にしてください。

```bash
$ gcloud compute ssh INSTANCE_NAME \
    --project PROJECT_NAME \
    --zone ZONE_NAME
```

## 4. どこにディスクのデータが存在するかを確認
InstanceにAttachされたdiskは`/dev/disk/by-id/`配下に`google-`の接頭語＋disk名の形でシンボリックリンクが作成されます。

```bash
me@instance:~$ ls -l /dev/disk/by-id/google-*
lrwxrwxrwx 1 root root  9 Jul 29 17:56 /dev/disk/by-id/google-persistent-disk-0 -> ../../sda
lrwxrwxrwx 1 root root 10 Jul 29 17:56 /dev/disk/by-id/google-persistent-disk-0-part1 -> ../../sda1
lrwxrwxrwx 1 root root  9 Aug 25 18:16 /dev/disk/by-id/google-temporary-disk -> ../../sdb
```

この場合は、`google-persistent-disk-0`がroot persistent disk、`google-temporary-disk`がtemporary diskです。

## 5. temporary diskをフォーマットしマウント
[safe_format_and_mount](https://github.com/GoogleCloudPlatform/compute-image-packages/blob/master/google-startup-scripts%2Fusr%2Fshare%2Fgoogle%2Fsafe_format_and_mount)を用いてtemporary diskのフォーマットとマウントを行います。

```bash
me@instance:~$ sudo mkdir /mnt/tmp
me@instance:~$ sudo /usr/share/google/safe_format_and_mount -m "mkfs.ext4 -F" /dev/sdb /mnt/tmp
```

4.で確認時にtemporary diskは`/dev/sdb`に存在することが確認できていますので、フォーマット対象には`/dev/sdb`を指定します。

safe_format_and_mountは[compute-image-packages](https://github.com/GoogleCloudPlatform/compute-image-packages)に含まれているシェルスクリプトでGCEにデフォルトで用意されているImageには予めインストールされています。もし存在しない場合には[wget](https://www.gnu.org/software/wget/)などを用いてcompute-image-packagesの[releases](https://github.com/GoogleCloudPlatform/compute-image-packages/releases)から最新のtar.gzファイルを取得すれば、利用することが出来ます。

## 6. カスタマイズしたroot persistent diskをtarファイル化
カスタマイズしたroot persistent diskをimage化しtarファイルに固めます。
この処理を実行するまでにImageに含めたいパッケージのインストールやファイルの配置を終わらせておいてください。

root persistent diskをtarファイルに固めるには[gcimagebundle](https://github.com/GoogleCloudPlatform/compute-image-packages/tree/master/gcimagebundle)を用います。
gcimagebundleもsafe_format_and_mountと同じく、compute-image-packagesに含まれるライブラリですので、基本的に予めインストールされています。

```bash
me@instance:~$ sudo gcimagebundle -d /dev/sda -o /mnt/tmp/ --log_file=/tmp/abc.log
```

上記コマンドを実行すると、次の場所にImageのtarファイルが作成されます。

```
/mnt/tmp/HEX-NUMBER.image.tar.gz
```

## 7. GCSへImageをアップロード
Instanceにプリインストールされている[gsutil](https://cloud.google.com/storage/docs/gsutil)コマンドラインツールを用いてGCSにファイルをアップロードします。

### 7.1 Bucketを作成
次のコマンドを用いて、Bucketを作成します。
Bucketの命名に関しては[Bucket and Object Naming Guidelines](https://cloud.google.com/storage/docs/bucketnaming?_ga=1.253453708.444133757.1413786301)を参考にしてください。

```bash
me@instance:~$ gsutil mb gs://BUCKET_NAME
```

### 7.2 BucketにImageをアップロード
次のコマンドを用いて、7.1で作成したBucketにImageのtarファイルをアップロードします。

```bash
me@instance:~$ gsutil cp /mnt/tmp/IMAGE_NAME.image.tar.gz gs://BUCKET_NAME
```

## おわりに
以上で、GCEからGCSにImageをアップロードする一覧の流れは終了です。
これを応用すれば、diskからだけでなく、SnapshotからInstanceを作成し、Image化してGCSにアップロードも可能です。

# このレポジトリについて

Surface Pro 3にUbuntu 16.04をインストールした時に、TypeCoverのキーボード部分のみが認識されなくなる問題を解決するためのカーネルパッチです。
TypeCoverのタッチパッド部分は認識されるが、キーボードのみ反応が無い場合はこのパッチの導入を試してみてください。

# この問題が発生するTypeCoverの見分け方

この問題は特定のTypeCoverのみで発生します。  
TypeCoverをSurfaceから取り外し、接着面の右端に薄く小さく印字された4桁の番号を確認してください。  
この番号が1709である場合は本パッチを適用することでキーボードが認識されない問題が解決するはずです。

番号が1644となっている場合は本パッチを当てる必要はありません。

# 前提

- Surface Pro 3にOSとしてUbuntu 16.04 (https://www.ubuntu.com/download/desktop)がインストールされている。 
- Ubuntuのkernelバージョンが4.4.0である。
- Ubuntuインストール直後である。
- /tmpに15GBほどの空き容量がある。

ここではSurface Pro 3にUbuntuをインストールする方法については書きません。  
インストール手順については下記のサイトを参考にしてください。

- https://webmake.info/surface-pro-3%e3%81%abubuntu-15-04%e3%82%92%e3%82%a4%e3%83%b3%e3%82%b9%e3%83%88%e3%83%bc%e3%83%ab%e3%81%97%e3%83%87%e3%83%a5%e3%82%a2%e3%83%ab%e3%83%96%e3%83%bc%e3%83%88%e3%81%99%e3%82%8b/

上記の手順ではTypeCover 1709のキーボードは認識されません。(1644のキーボードは問題無し)  
認識させるにはLinux kernelにパッチを当て、リビルドする必要があります。

# 参考
- [Ubuntu 16.04: Rebuild kernel with devscripts](https://www.hiroom2.com/2016/05/19/ubuntu-16-04-rebuild-kernel-with-devscripts/)
- [Notes, patches, etc for Ubuntu on Surface Pro 3](https://github.com/mar-io/surface-pro-3)

# 手順
0. 環境の確認

```
uname -r
```
...4.4.0-xx-generic となっていることを確認する。xxの部分は任意の数字に置き換える。

1. aptソースレポジトリの有効化

```
sudo vi /etc/apt/sources.list
```
...xenial-updatesのコメントアウトを外す。
```
  (before)
  # deb-src http://jp.archive.ubuntu.com/ubuntu/ xenial-updates main restricted
  (after)
  deb-src http://jp.archive.ubuntu.com/ubuntu/ xenial-updates main restricted
```

2. aptソースの更新
```
sudo apt update
sudo apt upgrade
```

3. ビルドに必要なパッケージのインストール
```
sudo apt install git build-essential kernel-package fakeroot libncurses5-dev libssl-dev ccache devscripts
```

4. ソースのダウンロード
```
uname -r
```
... kernelバージョンが4.4.0のままであることを確認する。
```
mkdir /tmp/new-kernel && cd /tmp/new-kernel
apt source linux-image-`uname -r`
sudo apt build-dep linux
```

5. パッチファイルのダウンロード
```
git clone https://github.com/Hinaser/SurfacePro3-Ubuntu
```

6. ソースビルドの前準備
```
cd linux-4.4.0
chmod -R u+x debian/scripts/*
fakeroot debian/rules distclean
fakeroot debian/rules debian/control
cp debian.master/changelog debian/
```

7. Changelogの編集
```
EDITOR=emacs debchange
```
... 下記を参考にChangelogを編集する。
```
linux (4.4.0-22.39ubuntu1) UNRELEASED; urgency=medium

  * Support Surface Pro 3 1709 TypeCover

 --  <hiroom2@ubuntu-16.04>  Mon, 16 May 2016 17:52:06 +0900
```

8. パッチの適用
```
patch -p1 --ignore-whitespace -i ../SurfacePro3-Ubuntu/Surface-Pro-3-TypeCover-1709.patch
```

9. Kernelビルド
```
DEB_BUILD_OPTIONS=parallel=2 do_tools=false no_dumpfile=1 fakeroot debian/rules binary-generic
```
...ビルド完了後、`ls`コマンドでdebファイルが作成されていることを確認する。
```
ls -l ../*.deb
```

10. Kernelのインストール
```
cd ../
sudo dpkg -i linux-image-*.deb linux-headers-*.deb
```
... /bootにビルドしたKernelがインストールされていることを確認する。
```
ls -l /boot/
```
... ※initrd.img-4.4.0-xx-genericが存在することを確認する。


11. Grub更新
```
sudo grub-install
```

12. X11設定の更新
```
vi /usr/share/X11/xorg.conf.d/10-evdev.conf
```
... 下記の行を追記する。
```
Section "InputClass"
    Identifier "Surface Pro 3 cover"
    MatchIsPointer "on"
    MatchDevicePath "/dev/input/event*"
    Driver "evdev"
    Option "vendor" "045e"
    Option "product" "07e3"
    Option "IgnoreAbsoluteAxes" "True"
EndSection
```

13. 再起動
```
sudo shutdown -r now
```

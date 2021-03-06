---
id: "2014-04-28-getting-started-with-fio"
title: "Getting Started With Fio"
abstract: "How to benchmark your storage with fio, the Flexible IO Tester"
tags: ["fio", "benchmark", "documentation"]
pubdate: 2014-04-28T22:00:00Z
---

Fio is easily the most powerful benchmarking tool available today. Because of its flexibility, it
has a reputation for being difficult to use. Actually, using it is pretty easy and reading the
output is hard, so I started with [explaining the output](/post/2014-04-17-fio-output-explained.html).

In order to run fio, you have to get fio. This is trivial on most Linux distributions.

    sudo pacman -S fio
    sudo apt-get install fio
    sudo yum install fio
    sudo emerge -av fio

It's also fairly easy on OSX if you use Homebrew, but does not appear to be available in Macports.
Installing from source is covered below.

    brew install fio

I did a quick search for Windows binaries of fio and found MSI installers offered at
[http://www.bluestop.org/fio/](http://www.bluestop.org/fio/).
[So far, so good](/images/fio-windows8.jpg)

And as always, you can build from source. You will need build tools installed, of course. If your
current distro has old fio packages, this might be the best way to go.

    git clone --branch fio-2.1.8 http://git.kernel.dk/fio.git fio-2.1.8
    # or
    wget http://brick.kernel.dk/snaps/fio-2.1.8.tar.bz2 && \
    tar -xjvf fio-2.1.8.tar.bz2

    cd fio-2.1.8
    ./configure
    make
    make install

Now that you have fio installed, it's time to run a benchmark. The first test runs 1 gigabyte of IO on a
subdirectory in $HOME. First create the test directory. Fio will create some files in this directory
and will perform all IO on files under it.

    mkdir $HOME/fio # Unix: this directory will written to by fio
    mkdir C:\fio    # Windows PowerShell

Next, create your configuration file. I've been calling this file simply 'trivial.fio'. In the Unix
example, I'm using the HOME environment variable to specify the IO path as ~/fio.

```
[global]
ioengine=posixaio
rw=readwrite
size=1g
directory=${HOME}/fio
thread=1

[trivial-readwrite-1g]
```

On Windows, I created the file using Notepad.
The ioengine needs to change to windowsaio or another engine supported on Windows
and the colon in the path must be escaped since fio uses it as a separator. Finally, tell fio to
use threads instead of processes since that's how Things Are Done on Windows. The Unix test was
switched to use threads on all platforms since it's needed on OSX as well.

```
[global]
ioengine=windowsaio
rw=readwrite
size=1g
directory=C\:\fio
thread=1

[trivial-readwrite-1g]
```

Finally, it's time to run the test. The command is the same on all platforms. You will need your
shell to be in the same directory as the trivial.fio file for this to work. All three of these
commands run the same benchmark, but differ in how output is delivered. You only need the first
one most of the time. The next two are useful if you want to save data for later comparison.

    fio trivial.fio
    fio trivial.fio --output=trivial.txt # write output to a file
    fio trivial.fio --output-format=json --output=trivial.json

And that's it. For comparison, I've uploaded the the output from some of my machines. A couple were
run in mmap mode before I switched to posixaio to keep closer to the Windows config.

* [Core i7-2600 / Samsung 840 EVO 240G SSD / Linux / btrfs](https://gist.github.com/tobert/11386257#file-brak-samsung840evo-linux-txt)
* [Xeon E31270 / Linux](/post/2014-03-29-benchmarking-disk-latency-setup.html)
    * SATA HBA
        * [2x Seagate ST9500530NS 500GB / btrfs RAID 1](https://gist.github.com/tobert/11386257#file-zorak-enterprise-sata-raid1-txt)
        * [Seagate ST31000340NS Enterprise 1TB / fuse / ntfs-3g](https://gist.github.com/tobert/11386257#file-zorak-enterprise-sata-1tb-ntfs-3g-txt)
        * [Seagate ST31000340NS Enterprise 1TB / ext4](https://gist.github.com/tobert/11386257#file-zorak-enterprise-sata-1tb-ext4-txt)
    * LSI SAS3004
        * [Seagate ST9500430SS 7200RPM 500GB / ext4](https://gist.github.com/tobert/11386257#file-zorak-7200rpm-sas2-txt)
        * [Samsung 840 Pro SSD 120GB / ext4](https://gist.github.com/tobert/11386257#file-brak-samsung840evo-linux-txt)
        * [Western Digital WD3000BLFS-0 10000RPM 300GB / ext4](https://gist.github.com/tobert/11386257#file-zorak-10krpm-sata-on-sas-hba-txt)
        * [PNY XLR8 SSD2SC240G1LC709 240GB / ext4](https://gist.github.com/tobert/11386257#file-zorak-pny-sata-ssd-on-sas-hba-txt)
    * PCI Express
        * [Fusion IO ioDrive2 1TB](https://gist.github.com/tobert/11386257#file-zorak-fusionio-iodriveii-txt)
        * [Raijin-256GB-MT](https://gist.github.com/tobert/11386257#file-zorak-raijin-pcie-256g-txt)
    * USB3
        * [Seagate ST500LM000 SSHD 5400RPM Laptop Hybrid / ext4](https://gist.github.com/tobert/11386257#file-zorak-usb3-sshd-5400rpm-ext4-txt)
    * USB2
        * [Generic 4GB Flash](https://gist.github.com/tobert/11386257#file-zorak-usb2-flash-4g-txt)
* 2013 Retina Macbook Pro (SSD)
    * [Windows 8.1 Pro / NTFS](https://gist.github.com/tobert/11386257#file-trivial-2013-mbp-win81pro-txt)
    * [OSX Mavericks / HFS+](https://gist.github.com/tobert/11386257#file-2013-macbook-pro-osx-mavericks-txt)

Now you've run your first benchmark with fio. Head over to
[Fio Output Explained](/post/2014-04-17-fio-output-explained.html) to find out what all those numbers
mean.

Keep in mind that this trivial test only does 1 gigabyte of IO. The numbers in the gist quoted above
should not be used to make any real-world decisions.

In my next post, I will be showing how to parse and plot the JSON output of fio.


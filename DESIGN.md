fast alpr with opencv-python-headless depenancy
https://github.com/ankandrew/fast-alpr

how to build opencv-python whell without fips mismatch problem
https://github.com/opencv/opencv-python/pull/1190
https://github.com/opencv/opencv-python/pull/1191

###

@AdityaMishra3000 , @asmorkalov , thank you for the fast responses. I am not used to this.
The #1190 PR would appear to resolve my issue.

I pulled your branch, built the wheel, and installed it.

[vmiller@gluskap tmp]$ git clone https://github.com/opencv/opencv-python.git
[vmiller@gluskap tmp]$ cd opencv-python
[vmiller@gluskap tmp]$ git fetch origin pull/1190/head:pr-1190
[vmiller@gluskap tmp]$ git checkout pr-1190
[vmiller@gluskap tmp]$ cd ~/opencv-python

[vmiller@gluskap opencv-python]$ uv venv
Using CPython 3.13.7
Creating virtual environment at: .venv
Activate with: source .venv/bin/activate
[vmiller@gluskap opencv-python]$ venv
(opencv-python) [vmiller@gluskap opencv-python]$ uv pip install scikit-build
-core numpy setuptools wheel cmake
Resolved 7 packages in 272ms
Prepared 5 packages in 897ms
Installed 7 packages in 49ms
 + cmake==4.2.1
 + numpy==2.4.1
 + packaging==26.0
 + pathspec==1.0.3
 + scikit-build-core==0.11.6
 + setuptools==80.10.2
 + wheel==0.46.3
(opencv-python) [vmiller@gluskap opencv-python]$ uv pip install scikit-build 
pip wheel . --no-build-isolation
Resolved 5 packages in 278ms
Prepared 2 packages in 67ms
Installed 2 packages in 13ms
 + distro==1.9.0
 + scikit-build==0.18.1
Processing /home/vmiller/Work/tmp/opencv-python
  Preparing metadata (pyproject.toml) ... done
Collecting numpy>=2 (from opencv-python==4.13.0+dc2e895)
  Using cached numpy-2.4.1-cp313-cp313-manylinux_2_27_x86_64.manylinux_2_28_x86_64.whl.metadata (6.6 kB)
Downloading numpy-2.4.1-cp313-cp313-manylinux_2_27_x86_64.manylinux_2_28_x86_64.whl (16.4 MB)
   ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━ 16.4/16.4 MB 52.7 MB/s  0:00:00
Saved ./numpy-2.4.1-cp313-cp313-manylinux_2_27_x86_64.manylinux_2_28_x86_64.whl
Building wheels for collected packages: opencv-python
  Building wheel for opencv-python (pyproject.toml) ... done
  Created wheel for opencv-python: filename=opencv_python-4.13.0+dc2e895-cp313-cp313-linux_x86_64.whl size=32287285 sha256=1dca8d48ad9dfcd1886f553468f7446af1971fa427a37656006c2a36eb88e42f
  Stored in directory: /home/vmiller/.cache/pip/wheels/b0/ab/eb/b0d579fc4ff1cefcdef823871b057fac80ff9d14819a19707d
Successfully built opencv-python
(opencv-python) [vmiller@gluskap opencv-python]$ cd ~

(opencv-python) [vmiller@gluskap opencv-python]$ uv pip install ./opencv_python*.whl
Resolved 2 packages in 116ms
Prepared 1 package in 165ms
Installed 1 package in 4ms
 + opencv-python==4.13.0+dc2e895 (from file:///home/vmiller/Work/tmp/opencv-python/opencv_python-4.13.0+dc2e895-cp313-cp313-linux_x86_64.whl)

(opencv-python) [vmiller@gluskap opencv-python]$ cd ~
(opencv-python) [vmiller@gluskap ~]$ python -c "import cv2; print(cv2.__version__)"
4.13.0
(opencv-python) [vmiller@gluskap ~]$ 

###

https://aikchar.dev/blog/add-fips-module-to-openssl-3011-on-debian-12-bookworm.html

OpenSSL 3.0 installed from deb package
$ apt update && apt install --no-install-recommends --no-install-suggests -y openssl
Locally built OpenSSL 3.0.9 from source code
These steps are simplified specific version of the process provided in Need steps to compile openssl-debian-openssl-3.0.11-1_deb12u2 with FIPS complaint openssl-3.0.8.

We build everything but only install the FIPS module.

$ curl -O https://www.openssl.org/source/old/3.0/openssl-3.0.9.tar.gz
$ tar xvzf openssl-3.0.9.tar.gz
$ cd openssl-3.0.9
$ ./Configure enable-fips --prefix=usr --openssldir=/usr/lib/ssl --libdir=lib/x86_64-linux-gnu
$ make
$ sudo make install_fips
Modified openssl.cnf to enable FIPS mode
$ sudo cp /usr/lib/ssl/fipsmodule.cnf /etc/ssl/fipsmodule.cnf
Create /etc/ssl/openssl-fips.cnf which will enable the FIPS module. It is a modified copy of the stock /etc/ssl/openssl.cnf file that comes with the openssl deb package.

$ sed \
    -e '51d' \
    -e '52i ##' \
    -e '52i ## https://www.openssl.org/docs/manmaster/man7/fips_module.html' \
    -e '52i ##' \
    -e '52i .include /etc/ssl/fipsmodule.cnf' \
    -e '54d' \
    -e '55i providers = provider_sect' \
    -e '55i alg_section = algorithm_sect' \
    -e '57d' \
    -e '58d' \
    -e '59i [provider_sect]' \
    -e '59i default = default_sect' \
    -e '59i \\n' \
    -e '61d' \
    -e '62i fips = fips_sect' \
    -e '71d' \
    -e '72d' \
    -e '73i [default_sect]' \
    -e '73i activate = 1' \
    -e '73i [algorithm_sect]' \
    -e '73i default_properties = fips=yes' \
    -e '73i \\n' \
    /etc/ssl/openssl.cnf | sudo tee /etc/ssl/openssl-fips.cnf
In the sed command above, we are inserting lines in specific places and removing specific lines. This is specific to the openssl deb package version 3.0.11-1 and may not work for other versions. This result could be achieved by regex search and replace as well but that is left up to you (for example, you can use the technique in Use awk Script to Modify a File).

Set environment variable OPENSSL_CONF
openssl uses environment variable OPENSSL_CONF to set the default configuration file. By default this variable is not set and in Debian the default file is /etc/ssl/openssl.cnf.

To switch to the FIPS config file we created above, set the environment variable,

$ export OPENSSL_CONF=/etc/ssl/openssl-fips.cnf
Verify FIPS is enabled
Check md5
MD5 will work when FIPS is not enabled,

$ openssl md5 /dev/null
MD5(/dev/null)= d41d8cd98f00b204e9800998ecf8427e
MD5 will not work when FIPS is enabled,

$ openssl md5 /dev/null
Error setting digest
Check supported ciphers
CHACHA20_POLY1305 is not supported in FIPS module. If it is present, openssl is not using FIPS.

For example, in the output below we see CHACHA20_POLY1305 is present, thus FIPS is not enabled,

$ openssl ciphers | tr ':' '\n' | sort
AES128-GCM-SHA256
AES128-SHA
AES128-SHA256
AES256-GCM-SHA384
AES256-SHA
AES256-SHA256
DHE-PSK-AES128-CBC-SHA
DHE-PSK-AES128-CBC-SHA256
DHE-PSK-AES128-GCM-SHA256
DHE-PSK-AES256-CBC-SHA
DHE-PSK-AES256-CBC-SHA384
DHE-PSK-AES256-GCM-SHA384
DHE-PSK-CHACHA20-POLY1305
DHE-RSA-AES128-GCM-SHA256
DHE-RSA-AES128-SHA
DHE-RSA-AES128-SHA256
DHE-RSA-AES256-GCM-SHA384
DHE-RSA-AES256-SHA
DHE-RSA-AES256-SHA256
DHE-RSA-CHACHA20-POLY1305
ECDHE-ECDSA-AES128-GCM-SHA256
ECDHE-ECDSA-AES128-SHA
ECDHE-ECDSA-AES128-SHA256
ECDHE-ECDSA-AES256-GCM-SHA384
ECDHE-ECDSA-AES256-SHA
ECDHE-ECDSA-AES256-SHA384
ECDHE-ECDSA-CHACHA20-POLY1305
ECDHE-PSK-AES128-CBC-SHA
ECDHE-PSK-AES128-CBC-SHA256
ECDHE-PSK-AES256-CBC-SHA
ECDHE-PSK-AES256-CBC-SHA384
ECDHE-PSK-CHACHA20-POLY1305
ECDHE-RSA-AES128-GCM-SHA256
ECDHE-RSA-AES128-SHA
ECDHE-RSA-AES128-SHA256
ECDHE-RSA-AES256-GCM-SHA384
ECDHE-RSA-AES256-SHA
ECDHE-RSA-AES256-SHA384
ECDHE-RSA-CHACHA20-POLY1305
PSK-AES128-CBC-SHA
PSK-AES128-CBC-SHA256
PSK-AES128-GCM-SHA256
PSK-AES256-CBC-SHA
PSK-AES256-CBC-SHA384
PSK-AES256-GCM-SHA384
PSK-CHACHA20-POLY1305
RSA-PSK-AES128-CBC-SHA
RSA-PSK-AES128-CBC-SHA256
RSA-PSK-AES128-GCM-SHA256
RSA-PSK-AES256-CBC-SHA
RSA-PSK-AES256-CBC-SHA384
RSA-PSK-AES256-GCM-SHA384
RSA-PSK-CHACHA20-POLY1305
SRP-AES-128-CBC-SHA
SRP-AES-256-CBC-SHA
SRP-RSA-AES-128-CBC-SHA
SRP-RSA-AES-256-CBC-SHA
TLS_AES_128_GCM_SHA256
TLS_AES_256_GCM_SHA384
TLS_CHACHA20_POLY1305_SHA256
In the output below CHACHA20_POLY1305 is not present, thus FIPS is enabled,

$ openssl ciphers | tr ':' '\n' | sort
AES128-GCM-SHA256
AES128-SHA
AES128-SHA256
AES256-GCM-SHA384
AES256-SHA
AES256-SHA256
DHE-PSK-AES128-CBC-SHA
DHE-PSK-AES128-CBC-SHA256
DHE-PSK-AES128-GCM-SHA256
DHE-PSK-AES256-CBC-SHA
DHE-PSK-AES256-CBC-SHA384
DHE-PSK-AES256-GCM-SHA384
DHE-RSA-AES128-GCM-SHA256
DHE-RSA-AES128-SHA
DHE-RSA-AES128-SHA256
DHE-RSA-AES256-GCM-SHA384
DHE-RSA-AES256-SHA
DHE-RSA-AES256-SHA256
ECDHE-ECDSA-AES128-GCM-SHA256
ECDHE-ECDSA-AES128-SHA
ECDHE-ECDSA-AES128-SHA256
ECDHE-ECDSA-AES256-GCM-SHA384
ECDHE-ECDSA-AES256-SHA
ECDHE-ECDSA-AES256-SHA384
ECDHE-PSK-AES128-CBC-SHA
ECDHE-PSK-AES128-CBC-SHA256
ECDHE-PSK-AES256-CBC-SHA
ECDHE-PSK-AES256-CBC-SHA384
ECDHE-RSA-AES128-GCM-SHA256
ECDHE-RSA-AES128-SHA
ECDHE-RSA-AES128-SHA256
ECDHE-RSA-AES256-GCM-SHA384
ECDHE-RSA-AES256-SHA
ECDHE-RSA-AES256-SHA384
PSK-AES128-CBC-SHA
PSK-AES128-CBC-SHA256
PSK-AES128-GCM-SHA256
PSK-AES256-CBC-SHA
PSK-AES256-CBC-SHA384
PSK-AES256-GCM-SHA384
RSA-PSK-AES128-CBC-SHA
RSA-PSK-AES128-CBC-SHA256
RSA-PSK-AES128-GCM-SHA256
RSA-PSK-AES256-CBC-SHA
RSA-PSK-AES256-CBC-SHA384
RSA-PSK-AES256-GCM-SHA384
SRP-AES-128-CBC-SHA
SRP-AES-256-CBC-SHA
SRP-RSA-AES-128-CBC-SHA
SRP-RSA-AES-256-CBC-SHA
TLS_AES_128_GCM_SHA256
TLS_AES_256_GCM_SHA384
Verify https ciphers
If you have configured a web server with TLS, for example nginx, you can also verify the ciphers it supports. If CHACHA20_POLY1305 is not present then FIPS is enabled.

In our example below, google.com has not enabled FIPS so we see CHACHA20_POLY1305.

$ nmap --script ssl-enum-ciphers -p 443 google.com
Starting Nmap 7.95 ( https://nmap.org ) at 2024-05-07 18:26 PDT
Nmap scan report for google.com (142.250.217.78)
Host is up (0.013s latency).
Other addresses for google.com (not scanned): 2607:f8b0:400a:80a::200e
rDNS record for 142.250.217.78: sea09s29-in-f14.1e100.net

PORT    STATE SERVICE
443/tcp open  https
| ssl-enum-ciphers:
|   TLSv1.0:
|     ciphers:
|       TLS_ECDHE_ECDSA_WITH_AES_128_CBC_SHA (ecdh_x25519) - A
|       TLS_ECDHE_ECDSA_WITH_AES_256_CBC_SHA (ecdh_x25519) - A
|       TLS_ECDHE_RSA_WITH_AES_128_CBC_SHA (ecdh_x25519) - A
|       TLS_ECDHE_RSA_WITH_AES_256_CBC_SHA (ecdh_x25519) - A
|       TLS_RSA_WITH_AES_128_CBC_SHA (rsa 2048) - A
|       TLS_RSA_WITH_AES_256_CBC_SHA (rsa 2048) - A
|       TLS_RSA_WITH_3DES_EDE_CBC_SHA (rsa 2048) - C
|     compressors:
|       NULL
|     cipher preference: server
|     warnings:
|       64-bit block cipher 3DES vulnerable to SWEET32 attack
|   TLSv1.1:
|     ciphers:
|       TLS_ECDHE_ECDSA_WITH_AES_128_CBC_SHA (ecdh_x25519) - A
|       TLS_ECDHE_ECDSA_WITH_AES_256_CBC_SHA (ecdh_x25519) - A
|       TLS_ECDHE_RSA_WITH_AES_128_CBC_SHA (ecdh_x25519) - A
|       TLS_ECDHE_RSA_WITH_AES_256_CBC_SHA (ecdh_x25519) - A
|       TLS_RSA_WITH_AES_128_CBC_SHA (rsa 2048) - A
|       TLS_RSA_WITH_AES_256_CBC_SHA (rsa 2048) - A
|       TLS_RSA_WITH_3DES_EDE_CBC_SHA (rsa 2048) - C
|     compressors:
|       NULL
|     cipher preference: server
|     warnings:
|       64-bit block cipher 3DES vulnerable to SWEET32 attack
|   TLSv1.2:
|     ciphers:
|       TLS_ECDHE_ECDSA_WITH_AES_128_CBC_SHA (ecdh_x25519) - A
|       TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256 (ecdh_x25519) - A
|       TLS_ECDHE_ECDSA_WITH_AES_256_CBC_SHA (ecdh_x25519) - A
|       TLS_ECDHE_ECDSA_WITH_AES_256_GCM_SHA384 (ecdh_x25519) - A
|       TLS_ECDHE_ECDSA_WITH_CHACHA20_POLY1305_SHA256 (ecdh_x25519) - A
|       TLS_ECDHE_RSA_WITH_AES_128_CBC_SHA (ecdh_x25519) - A
|       TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256 (ecdh_x25519) - A
|       TLS_ECDHE_RSA_WITH_AES_256_CBC_SHA (ecdh_x25519) - A
|       TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384 (ecdh_x25519) - A
|       TLS_ECDHE_RSA_WITH_CHACHA20_POLY1305_SHA256 (ecdh_x25519) - A
|       TLS_RSA_WITH_3DES_EDE_CBC_SHA (rsa 2048) - C
|       TLS_RSA_WITH_AES_128_CBC_SHA (rsa 2048) - A
|       TLS_RSA_WITH_AES_128_GCM_SHA256 (rsa 2048) - A
|       TLS_RSA_WITH_AES_256_CBC_SHA (rsa 2048) - A
|       TLS_RSA_WITH_AES_256_GCM_SHA384 (rsa 2048) - A
|     compressors:
|       NULL
|     cipher preference: client
|     warnings:
|       64-bit block cipher 3DES vulnerable to SWEET32 attack
|   TLSv1.3:
|     ciphers:
|       TLS_AKE_WITH_AES_128_GCM_SHA256 (ecdh_x25519) - A
|       TLS_AKE_WITH_AES_256_GCM_SHA384 (ecdh_x25519) - A
|       TLS_AKE_WITH_CHACHA20_POLY1305_SHA256 (ecdh_x25519) - A
|     cipher preference: client
|_  least strength: C

Nmap done: 1 IP address (1 host up) scanned in 1.62 seconds
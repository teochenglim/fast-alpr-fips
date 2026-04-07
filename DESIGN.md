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


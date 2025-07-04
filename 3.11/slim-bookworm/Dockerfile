#
# NOTE: THIS DOCKERFILE IS GENERATED VIA "apply-templates.sh"
#
# PLEASE DO NOT EDIT IT DIRECTLY.
#

FROM debian:bookworm-slim

RUN set -eux; \
	apt-get update; \
	apt-get install -y --no-install-recommends ca-certificates; \
	rm -rf /var/lib/apt/lists/*

# http://bugs.python.org/issue19846
# > At the moment, setting "LANG=C" on a Linux system *fundamentally breaks Python 3*, and that's not OK.
ENV LANG C.UTF-8

# ensure local pypy3 is preferred over distribution pypy3
ENV PATH /opt/pypy/bin:$PATH

# Python 3.11.13
ENV PYPY_VERSION 7.3.20

RUN set -eux; \
	\
	dpkgArch="$(dpkg --print-architecture)"; \
	case "${dpkgArch##*-}" in \
		'amd64') \
			url='https://downloads.python.org/pypy/pypy3.11-v7.3.20-linux64.tar.bz2'; \
			sha256='1410db3a7ae47603e2b7cbfd7ff6390b891b2e041c9eb4f1599f333677bccb3e'; \
			;; \
		'arm64') \
			url='https://downloads.python.org/pypy/pypy3.11-v7.3.20-aarch64.tar.bz2'; \
			sha256='9347fe691a07fd9df17a1b186554fb9d9e6210178ffef19520a579ce1f9eb741'; \
			;; \
		'i386') \
			url='https://downloads.python.org/pypy/pypy3.11-v7.3.20-linux32.tar.bz2'; \
			sha256='d08ce15dd61e9ace5e010b047104f0137110a258184e448ea8239472f10cf99b'; \
			;; \
		*) echo >&2 "error: current architecture ($dpkgArch) does not have a corresponding PyPy $PYPY_VERSION binary release"; exit 1 ;; \
	esac; \
	\
	savedAptMark="$(apt-mark showmanual)"; \
	apt-get update; \
	apt-get install -y --no-install-recommends \
		bzip2 \
		wget \
# sometimes "pypy3" itself is linked against libexpat1 / libncurses5, sometimes they're ".so" files in "/opt/pypy/lib/pypy3.11"
		libexpat1 \
		libncurses5 \
		libncursesw6 \
		libsqlite3-0 \
# (so we'll add them temporarily, then use "ldd" later to determine which to keep based on usage per architecture)
	; \
	\
	wget -O pypy.tar.bz2 "$url" --progress=dot:giga; \
	echo "$sha256 *pypy.tar.bz2" | sha256sum --check --strict -; \
	mkdir /opt/pypy; \
	tar -xjC /opt/pypy --strip-components=1 -f pypy.tar.bz2; \
	find /opt/pypy/lib* -depth -type d -a \( -name test -o -name tests \) -exec rm -rf '{}' +; \
	rm pypy.tar.bz2; \
	\
	ln -sv '/opt/pypy/bin/pypy3' /usr/local/bin/; \
	\
# smoke test
	pypy3 --version; \
	\
	cd /opt/pypy/lib/pypy3.11; \
# on pypy3, rebuild gdbm ffi bits for compatibility with Debian Stretch+
	if [ -f _gdbm_build.py ]; then \
		apt-get install -y --no-install-recommends gcc libc6-dev libgdbm-dev; \
		pypy3 _gdbm_build.py; \
	fi; \
# https://github.com/docker-library/pypy/issues/24#issuecomment-409408657
	if [ -f _ssl_build.py ]; then \
		apt-get install -y --no-install-recommends gcc libc6-dev libssl-dev; \
		pypy3 _ssl_build.py; \
	fi; \
# https://github.com/docker-library/pypy/issues/42
	if [ -f _lzma_build.py ]; then \
		apt-get install -y --no-install-recommends gcc libc6-dev liblzma-dev; \
		pypy3 _lzma_build.py; \
	fi; \
# https://github.com/docker-library/pypy/issues/68
	if [ -f _sqlite3_build.py ]; then \
		apt-get install -y --no-install-recommends gcc libc6-dev libsqlite3-dev; \
		pypy3 _sqlite3_build.py; \
	fi; \
# TODO rebuild other cffi modules here too? (other _*_build.py files)
	\
	apt-mark auto '.*' > /dev/null; \
	[ -z "$savedAptMark" ] || apt-mark manual $savedAptMark > /dev/null; \
	find /opt/pypy -type f -executable -exec ldd '{}' ';' \
		| awk '/=>/ { so = $(NF-1); if (index(so, "/usr/local/") == 1) { next }; gsub("^/(usr/)?", "", so); printf "*%s\n", so }' \
		| sort -u \
		| xargs -r dpkg-query --search \
		| cut -d: -f1 \
		| sort -u \
		| xargs -r apt-mark manual \
	; \
	apt-get purge -y --auto-remove -o APT::AutoRemove::RecommendsImportant=false; \
	rm -rf /var/lib/apt/lists/*; \
# smoke test again, to be sure
	pypy3 --version; \
	\
	pypy3 -m ensurepip --default-pip; \
	pip --version; \
# https://github.com/docker-library/python/issues/952
# https://github.com/docker-library/python/issues/1023
	pip install --disable-pip-version-check --no-cache-dir --no-compile 'wheel<0.46'; \
	\
	find /opt/pypy -depth \
		\( \
			\( -type d -a \( -name test -o -name tests \) \) \
			-o \
			\( -type f -a \( -name '*.pyc' -o -name '*.pyo' \) \) \
		\) -exec rm -rf '{}' +

CMD ["pypy3"]

# build-helper

Common script used in other build scripts.

- `host-functions-source.sh`: to be included with `source` in the host build script
- `container-functions-source.sh`: to be included with `source` in the container build script

Deprecated:

- `builder-helper.sh`: used in the first generation of build scripts.

## Patches

The code used to download and extract archives can also be used
to post-patch the downloaded files. For this a patch file must be
placed in the `patches` folder, and the name must be passed as the
third param to `extract()`.

## Memo

The code to apply the patch (`common-functions-source.sh:extract()`) does
the following:

```sh
cd sources/binutils

patch -p0 < "binutils-2.31.patch"
```

The patch is applied from the library source folder, so it must be created
with the library relative path.

For example, to create a binutils patch, use:

```sh
cd sources/binutils

cp bfd/ihex.c bfd/ihex-patched.c
vi bfd/ihex-patched.c
diff -u bfd/ihex.c bfd/ihex-patched.c > patches/binutils-2.31.patch
```

Alternatively, multiple changes can be generated by Git:

```sh
git init
git add -A
git commit -m all
```

Perform changes as usual.

To create a patch with the uncommitted changes that will be consumed
with `patch`:

```sh
git diff --no-prefix -u > file.patch
```

For completeness, to create a patch with the uncommitted changes that
will be consumeed with `git apply file.git-patch`.

```sh
git diff > file.git-patch
```

It is also possible to create patches with git and consume them with patch

```sh
git diff --no-prefix -u > file.patch
```

## GitHub archives

Normally the URL uses the  look like:

```sh
  local libxxx_archive="${libxxx_src_folder_name}.tar.gz"
  local libxxx_url="https://ftp.gnu.org/gnu/libxxx/${libxxx_archive}"

    download_and_extract "${libxxx_url}" "${libxxx_archive}" \
      "${libxxx_src_folder_name}"

```

For projects which download GitHub releases or even tags:

```sh
function build_libxxx()
{
  # https://github.com/Homebrew/homebrew-core/blob/master/Formula/libxxx.rb

  local libxxx_version="$1"

  local libxxx_src_folder_name="libxxx-${libxxx_version}"

  local libxxx_archive="${libxxx_src_folder_name}.tar.gz"
  # GitHub release archive.
  local libxxx_github_archive="v${libxxx_version}.tar.gz"
  local libxxx_github_url="https://github.com/libxxx/libxxx/archive/${libxxx_github_archive}"

  local libxxx_folder_name="${libxxx_src_folder_name}"

  mkdir -pv "${LOGS_FOLDER_PATH}/${libxxx_folder_name}"

  local libxxx_stamp_file_path="${INSTALL_FOLDER_PATH}/stamp-${libxxx_folder_name}-installed"
  if [ ! -f "${libxxx_stamp_file_path}" ]
  then

    cd "${SOURCES_FOLDER_PATH}"

    download_and_extract "${libxxx_github_url}" "${libxxx_archive}" \
      "${libxxx_src_folder_name}"

    (
      mkdir -pv "${LIBS_BUILD_FOLDER_PATH}/${libxxx_folder_name}"
      cd "${LIBS_BUILD_FOLDER_PATH}/${libxxx_folder_name}"

      xbb_activate_installed_dev

      CPPFLAGS="${XBB_CPPFLAGS}"
      CFLAGS="${XBB_CFLAGS_NO_W}"
      CXXFLAGS="${XBB_CXXFLAGS_NO_W}"
      LDFLAGS="${XBB_LDFLAGS_LIB}"

      if [ "${TARGET_PLATFORM}" == "linux" ]
      then
        LDFLAGS+=" -Wl,-rpath,${LD_LIBRARY_PATH}"
      fi

      export CPPFLAGS
      export CFLAGS
      export CXXFLAGS
      export LDFLAGS

      if [ ! -f "config.status" ]
      then
        (
          if [ "${IS_DEVELOP}" == "y" ]
          then
            env | sort
          fi

          echo
          echo "Running libxxx configure..."

          if [ "${IS_DEVELOP}" == "y" ]
          then
            run_verbose bash "${SOURCES_FOLDER_PATH}/${libxxx_src_folder_name}/configure" --help
          fi

          config_options=()

          config_options+=("--prefix=${LIBS_INSTALL_FOLDER_PATH}")

          config_options+=("--build=${BUILD}")
          config_options+=("--host=${HOST}")
          config_options+=("--target=${TARGET}")

          run_verbose bash ${DEBUG} "${SOURCES_FOLDER_PATH}/${libxxx_src_folder_name}/configure" \
            "${config_options[@]}"

          cp "config.log" "${LOGS_FOLDER_PATH}/${libxxx_folder_name}/config-log-$(ndate).txt"
        ) 2>&1 | tee "${LOGS_FOLDER_PATH}/${libxxx_folder_name}/configure-output-$(ndate).txt"
      fi

      (
        echo
        echo "Running libxxx make..."

        # Build.
        run_verbose make -j ${JOBS}

        if [ "${WITH_TESTS}" == "y" ]
        then
          run_verbose make -j1 check
        fi

        if [ "${WITH_STRIP}" == "y" ]
        then
          run_verbose make install-strip
        else
          run_verbose make install
        fi

      ) 2>&1 | tee "${LOGS_FOLDER_PATH}/${libxxx_folder_name}/make-output-$(ndate).txt"

      copy_license \
        "${SOURCES_FOLDER_PATH}/${libxxx_src_folder_name}" \
        "${libxxx_folder_name}"

    )

    touch "${libxxx_stamp_file_path}"

  else
    echo "Library libxxx already installed."
  fi

```

## Links

- <https://git-scm.com/docs/git-diff>
- <https://git-scm.com/docs/git-apply>

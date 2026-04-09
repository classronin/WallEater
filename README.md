# WallEater —— 吞墙者
在不使用传统代理的情况下绕过网络限制。适用于 Scoop、vcpkg 等的配方。
---

### [vcpkg](https://github.com/microsoft/vcpkg) 设置镜像
路径：
```
...\vcpkg\scripts\cmake\vcpkg_download_distfile.cmake 
```
```
function(vcpkg_download_distfile out_var)
    cmake_parse_arguments(PARSE_ARGV 1 arg
        "SKIP_SHA512;SILENT_EXIT;QUIET;ALWAYS_REDOWNLOAD"
        "FILENAME;SHA512"
        "URLS;HEADERS"
    )

    if(NOT DEFINED arg_URLS)
        message(FATAL_ERROR "vcpkg_download_distfile requires a URLS argument.")
    endif()
    if(NOT DEFINED arg_FILENAME)
        message(FATAL_ERROR "vcpkg_download_distfile requires a FILENAME argument.")
    endif()
    if(arg_SILENT_EXIT)
        message(WARNING "SILENT_EXIT no longer has any effect. To resolve this warning, remove SILENT_EXIT.")
    endif()

    # ========== 在这里添加镜像转换代码 ==========
    # 为所有 GitHub URL 添加镜像前缀
    set(mirror_prefix "https://gh-proxy.com/")
    set(converted_urls "")
    foreach(url IN LISTS arg_URLS)
        if(url MATCHES "^https://github\.com/")
            # 将 https://github.com/xxx 替换为 https://gh-proxy.com/https://github.com/xxx
            string(REPLACE "https://github.com/" "${mirror_prefix}https://github.com/" new_url "${url}")
            list(APPEND converted_urls "${new_url}")
            message(STATUS "Converted GitHub URL: ${url} -> ${new_url}")
        else()
            list(APPEND converted_urls "${url}")
        endif()
    endforeach()
    set(arg_URLS ${converted_urls})
    # ========== 镜像转换代码结束 ==========


    # Note that arg_ALWAYS_REDOWNLOAD implies arg_SKIP_SHA512, and NOT arg_SKIP_SHA512 implies NOT arg_ALWAYS_REDOWNLOAD
    if(arg_ALWAYS_REDOWNLOAD AND NOT arg_SKIP_SHA512)
        message(FATAL_ERROR "ALWAYS_REDOWNLOAD requires SKIP_SHA512")
```
---

### [Scoop](https://github.com/ScoopInstaller/Scoop/) 改造
- 效仿 vcpkg 实现镜像加速
- 修改 manifest.ps1 脚本替换镜像
```
%SCOOP%\apps\scoop\current\lib\manifest.ps1
```
manifest.ps1 中有一个非常简洁的 url 函数：
```
function url($manifest, $arch) { arch_specific 'url' $manifest $arch }
```
- 替换
```
function url($manifest, $arch) {
    $rawUrl = arch_specific 'url' $manifest $arch
    if (-not $rawUrl) { return $null }

    # ========== 镜像加速 ==========
    $mirror_prefix = "https://gh-proxy.com/"
    $mirror_domains = @("github.com", "raw.githubusercontent.com", "objects.githubusercontent.com")

    # 处理数组或字符串
    if ($rawUrl -is [Array]) {
        $processedUrls = @()
        foreach ($singleUrl in $rawUrl) {
            $newUrl = $singleUrl
            foreach ($domain in $mirror_domains) {
                if ($singleUrl -match "https?://(www\.)?$domain/") {
                    $newUrl = $mirror_prefix + $singleUrl
                    Write-Host "镜像: $newUrl" -ForegroundColor Cyan
                    break
                }
            }
            $processedUrls += $newUrl
        }
        return $processedUrls
    } else {
        $newUrl = $rawUrl
        foreach ($domain in $mirror_domains) {
            if ($rawUrl -match "https?://(www\.)?$domain/") {
                $newUrl = $mirror_prefix + $rawUrl
                Write-Host "镜像: $newUrl" -ForegroundColor Cyan
                break
            }
        }
        return $newUrl
    }
    # ==============================
}
```
---

### [Aria2](https://github.com/aria2/aria2) 安装并启用激进
```
scoop install main/aria2
scoop config aria2-enabled true
scoop config aria2-max-connection-per-server 16
scoop config aria2-split 32
scoop config aria2-min-split-size 1M
scoop config aria2-options "--async-dns=false --max-concurrent-downloads=5 --lowest-speed-limit=0 --disk-cache=32M --file-allocation=falloc"

验证
scoop config
```




# Anonymous GitHub account mass-dropping undisclosed 0-days

今日有新内容上线 ;) 迄今为止最重磅的一波
如果你想与我合作/交流，请在 Discord 上联系我 @ashdfrkl
分享这个仓库能让我有动力继续向大家发布我的研究发现。

这是我公开的概念验证（PoC）和漏洞研究文章的整合归档。
大多数文件夹都包含我此前某个独立的 PoC 仓库，并以其原始 README 和跟踪文件的形式予以保留。新的研究条目则作为自包含的文件夹直接添加在这里。

| 文件夹 | 来源 | 跟踪条目数 |
|---|---|---|
7zip-rar5-motw-chain-poc |
bd9533f532c1e4ee6af783b9bb49d1133c600e2c |
3 |
anydesk-printer-com-impersonation-poc |
7491303301093b2d40bee9dadf6b38f757ce78e0 |
4 |
c-ares-tcp-uaf-calc-poc |
直接条目，2026 年 6 月 24 日 | 7 |
docker-cp-copyout-destination-escape |
d1367b1381736d7f961ac808ce88d4e24a633adc |
5 |
firefox-smartwindow-private-url-exfil-poc |
直接条目，2026 年 6 月 24 日 | 3 |
floci-apigateway-vtl-rce-poc |
直接条目，2026 年 6 月 23 日 | 3 |
flowise-mcp-env-case-bypass-poc |
ed9fab0086674f1b16467990b33bb9299e93429e |
3 |
ffmpeg-rasc-dlta-calc-poc |
直接条目，2026 年 6 月 26 日 | 7 |
ghidra-12.1.2-rce-ace-calc-poc |
52dee6362990c03c0d753d074c85428824d46368 |
9 |
gitea-act-runner-container-options-poc |
f06d78fb111732f3e7737f4c07e77ef94c4b64bf |
4 |
imagemagick-gs-delegate-hijack-poc |
8140e8ee0ed78beaf5e8303a795b70b138f5891b |
5 |
libssh2-cve-2026-55200-poc |
直接条目，2026 年 6 月 23 日 | 3 |
libssh2-publickey-list-calc-poc |
直接条目，2026 年 6 月 25 日 | 10 |
lunar-modrinth-chain-poc |
ffd02120708b6503f11585858ce3724872f3b7a7 |
6 |
mybb-limited-acp-to-admin |
1610e0373943c2f6562a99f917d3a3d1fdd9056d |
5 |
nghttp2-nghttpx-upgrade-queue-poison-poc |
直接条目，2026 年 6 月 26 日 | 3 |
nmap-ipv6-extlen-wrap-poc |
直接条目，2026 年 6 月 23 日 | 4 |
objdump-dlx-calc-poc |
7df01e4e20c7375a89e8ccf760526c52eb6ad582 |
41 |
openvpn-connect-echo-script-ace-poc |
d2f904d9272d4388c9862131d40e32e072e85e38 |
8 |
php857-streambucket-soap-rce-rpoc |
直接条目，2026 年 6 月 26 日 | 6 |
rustdesk-session-permission-pocs |
直接条目，2026 年 6 月 25 日 | 17 |
systeminformer-phsvc-trusted-host-lpe-poc |
直接条目，2026 年 6 月 24 日 | 3 |
vlc-vp9-reschange-crash-poc |
fae72b82f24d03cf2fb9cb55fbb2e7774f684ff3 |
3 |

本节适用于上文按提交哈希列出的former（前）独立仓库。
本次整合是在旧的独立仓库被移除之前，于 2026 年 6 月 23 日从全新的 GitHub 克隆中进行检查的。
该检查将每个former独立仓库的 HEAD
树与本仓库中对应的文件夹进行比对，使用的是 Git 树数据，而非松散的文件系统差异比对。对于每一个跟踪条目，检查均要求：
- 相同的相对路径；
- 相同的 Git 对象类型；
- 相同的树模式（mode），包括可执行位；
- 相同的 Git blob ID。

Git blob ID 相同意味着被跟踪文件的字节内容完全一致。本次检查覆盖了 12 个仓库和 96 个跟踪条目，零不匹配。
本仓库保留了这些 PoC 的内容。仓库层面的元数据，例如 star 数、issues、pull requests、releases 以及独立的 Git 历史，仍保留在原始仓库的历史记录中。

直接条目，包括 c-ares-tcp-uaf-calc-poc
、ffmpeg-rasc-dlta-calc-poc
、firefox-smartwindow-private-url-exfil-poc
、floci-apigateway-vtl-rce-poc
、libssh2-cve-2026-55200-poc
、libssh2-publickey-list-calc-poc
、nghttp2-nghttpx-upgrade-queue-poison-poc
、nmap-ipv6-extlen-wrap-poc
、php857-streambucket-soap-rce-rpoc
、rustdesk-session-permission-pocs
以及 systeminformer-phsvc-trusted-host-lpe-poc
，均由本仓库的提交历史进行跟踪。

请勿在任何情况下恶意使用本仓库中的任何材料。这是善意的、公开披露的漏洞研究，旨在吸引更多人探索网络安全的这一领域。
网络犯罪很逊（cringe）。

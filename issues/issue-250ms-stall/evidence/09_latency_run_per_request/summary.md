# 單筆查詢延遲拆解（單一連線、依序送出、無其他負載）

| 項目 | rhds13 |
|---|---:|
| 樣本數 | 2000 |

## 拆解（中位數 p50，毫秒；各段相加約等於 ldap3 查詢的 p50）

| 項目 | rhds13 |
|---|---:|
| ① 網路 RTT（ICMP ping） | 0.270 |
| ② Server 記錄時間（etime） | 0.207 |
| 　└ 排隊（RHDS wtime / OpenLDAP qtime） | 0.087 |
| 　└ 處理（RHDS optime / OpenLDAP etime − qtime） | 0.119 |
| ③ Server 未記錄時間（wire − ① − ②） | -0.030 |
| ④ Client 函式庫（ldap3 − wire） | 0.719 |
| **wire 往返（① + ② + ③）** | 0.447 |
| **ldap3 查詢（① + ② + ③ + ④）** | 1.166 |

## 各階段分佈（毫秒）

| 項目 | rhds13 |
|---|---:|
| ICMP ping p50 / p95 / p99 / 平均 | 0.270 / 0.355 / 0.497 / 0.319 |
| ICMP ping 慢操作（> 10 ms）筆數 / 最大值 | 0 / 4.590 |
| TCP 連線建立（參考） p50 / p95 / p99 / 平均 | 0.408 / 0.560 / 0.813 / 0.422 |
| TCP 連線建立（參考） 慢操作（> 10 ms）筆數 / 最大值 | 0 / 0.935 |
| TLS handshake（參考） p50 / p95 / p99 / 平均 | 4.702 / 6.547 / 8.648 / 4.950 |
| TLS handshake（參考） 慢操作（> 10 ms）筆數 / 最大值 | 0 / 8.648 |
| wire 往返 p50 / p95 / p99 / 平均 | 0.447 / 0.664 / 1.172 / 1.225 |
| wire 往返 慢操作（> 10 ms）筆數 / 最大值 | 6 / 251.284 |
| ldap3 查詢 p50 / p95 / p99 / 平均 | 1.166 / 2.074 / 3.212 / 2.164 |
| ldap3 查詢 慢操作（> 10 ms）筆數 / 最大值 | 7 / 251.708 |
| Server etime p50 / p95 / p99 / 最大（wire 階段） | 0.207 / 0.292 / 0.464 / 5.722 |
| Server log 對應筆數（wire / ldap3） | 2008 / 2000 |
| 查詢異常（wire / ldap3） | 0 / 0 |

## 說明

- 拆解使用中位數；少數停頓（慢操作）會大幅拉高平均值，另外列出筆數與最大值。若慢操作的 Server etime 很小，表示停頓發生在 Server 記錄範圍之外。
- 網路 RTT 以 ICMP ping 代表（兩端都由 kernel 回應，不經 LDAP Server 程序）。TCP 連線建立僅供參考：新連線需經 Server 端 conntrack 與防火牆規則，會高於已建立連線的往返。
- wire：自行編碼的 BER SearchRequest 直接經 TLS socket 送出，收到 SearchResultDone 即停止，不含 ldap3 成本。
- Server 記錄時間：RHDS access log 的 etime（= wtime + optime）；OpenLDAP stats log 的 etime（含 qtime）。兩者都從 Server 讀到完整操作開始計時，不含 TLS 解密前、結果送出後的時間。
- Server 未記錄時間：TLS 加解密、listener / I/O thread 收送、操作排入佇列前與結果寫出後的處理。
- 量測期間兩套產品都開啟操作層級的 log（RHDS access log、OpenLDAP stats），結束後還原。

# Lab 25B - Report

**Ho ten / nhom:** Vu Van Ha - 202602589  **Commit:** a4facfd  **Diem tu dong (`make grade`):** 84.0 / 88

## 1. Ket Qua Game Day

| Bang | Client | availability | p95 (s) | amplification | retry_ratio | cost/turn | recovery (s) | PASS? |
|---|---|---:|---:|---:|---:|---:|---:|:---:|
| G1 | naive | 0.7194 | 7.157 | 7.738 | 6.738 | 2.163 | 4.0 | no |
| G1 | yours | 1.0000 | 4.776 | 1.062 | 0.002 | 2.101 | 0.0 | yes |
| G2 | naive | 0.4560 | 15.034 | 9.904 | 8.904 | 4.627 | 155.0 | no |
| G2 | yours | 1.0000 | 10.920 | 1.041 | 0.007 | 2.119 | 0.0 | yes |
| G3 | naive | 1.0000 | 4.768 | 1.164 | 0.164 | 2.328 | 0.0 | no |
| G3 | yours | 1.0000 | 4.660 | 1.060 | 0.034 | 2.145 | 0.0 | yes |
| G4 | naive | 1.0000 | 13.764 | 1.004 | 0.004 | 2.008 | 0.0 | no |
| G4 | yours | 1.0000 | 5.562 | 1.015 | 0.002 | 2.072 | 0.0 | yes |
| G5 | naive | 0.3699 | 5.359 | 17.030 | 16.030 | 2.124 | n/a | no |
| G5 | yours | 0.8179 | 5.477 | 1.003 | 0.002 | 2.504 | n/a | yes |
| G6 | naive | 0.5170 | 4.836 | 12.344 | 11.344 | 2.009 | 0.0 | no |
| G6 | yours | 1.0000 | 5.961 | 1.055 | 0.019 | 2.189 | 0.0 | yes |

## 2. Cau Hinh Client

| Tham so | Gia tri | Ly do |
|---|---:|---|
| attempt_timeout_s | 8.0 | Du ngan de failover truoc deadline 30s; G6 dat availability 1.0000, p95 5.961s. |
| max_attempts_per_provider | 2 | Chi 1 retry cung provider, G2 amplification giam tu 9.904 xuong 1.041. |
| backoff_base_s / cap_s | 0.25 / 4.0 | Jitter nho de tranh dong bo retry; G1 co 0 retry-after violations. |
| retry_budget_ratio | 0.1 | G3 retry_ratio 0.034, duoi nguong 0.10. |
| breaker.window_size / min_calls | 20 / 10 | Du mau de khong trip vi loi le, van bat G2/G4 nhanh. |
| breaker.failure_rate_threshold | 0.5 | Khi overload/503/timeout qua nua window thi failover; G2 recovery 0.0s. |
| breaker.slow_call_s | 5.0 | Bat slow region truoc khi timeout; G4 p95 giam 13.764s -> 5.562s. |
| breaker.open_duration_s | 10.0 | Cho provider ha nhiet nhung probe lai som; primary_share_after_restore G2/G4 = 1.0. |
| breaker.half_open_max_probes | 2 | Gioi han thundering herd khi recover; amplification cac bang <= 1.062. |

Bang quyet dinh `on_error`: client error -> release breaker, give up; quota -> release, cooldown 300s, failover; rate limit -> release, cooldown theo retry-after, failover; overload/server/timeout -> ghi breaker failure, retry cung provider neu con budget va kip deadline, neu khong thi failover.

## 3. Ba Thi Nghiem

Retry budget la lop phong thu cuoi: trong G2 naive amplification 9.904 va retry_ratio 8.904, con client co budget/breaker chi 1.041 va 0.007. G3 naive retry_ratio 0.164 vuot nguong, ban moi la 0.034. Breaker dung giup cat nhanh provider xau, budget van can de neu breaker cau hinh sai thi retry khong bien thanh storm.

`attempt_timeout_s` khong nen dat 30s vi G6 co provider treo: neu cho moi attempt giu tron deadline, call khong con thoi gian failover. Cau hinh 8s giu G6 availability 1.0000; G4 gan nhu khong can retry ma can breaker slow-call de p95 tu 13.764s xuong 5.562s. Timeout nen dua vao p99 binh thuong, deadline user va thoi gian can de thu provider tiep theo.

Bo loc tool calling se day tool call sang `gamma-mini`, model khong ho tro tool. Trong G6 ban hien tai co `incompatible_routes = 0`; ve mat san pham, day la tranh tra loi sai quy trinh hoan tien chi vi fallback sang model re hon.

## 4. Workflow Hoan Tien

| Executor | completion | crashes | duplicate refunds | duplicate emails | lost refunds | overpaid | wasted LLM / crash |
|---|---:|---:|---:|---:|---:|---:|---:|
| naive | 1.0000 | 81 | 128 | 81 | 0 | 1496700 | 1.123 |
| yours | 1.0000 | 66 | 0 | 0 | 0 | 0 | 0.303 |

Chi idempotency key chua du vi restart van phai tinh lai cac step thuong, gay lang phi LLM. Chi checkpoint chua du vi crash/timeout sau payment co the chay lai side effect neu khong dung cung key. Neu ghi checkpoint truoc payment, worker restart se tuong tien da di va co the mat refund.

## 5. Alerting Va Error Budget

| Su co | Mong doi | Naive alerter | Burn-rate alerter | Do tre |
|---|---|---|---|---:|
| checkout_bug | page | page | page | 27 |
| provider_outage | page | page | page | 4 |
| deploy_blip | none | false page | none | n/a |
| slow_leak | ticket | missed | ticket | 1337 |
| degraded_evening | page | page | page | 291 |
| night_spike | none | false page | none | n/a |
| small_bump | none | false page | none | n/a |

Error budget 30 ngay: 129508.8 bad requests duoc phep, da tieu 55618 = 42.95%, con 57.05%, nen khong freeze release. `small_bump` khong nen page vi short blip khong dot du ca cua so dai va ngan; page chi danh cho burn dang tiep dien va co tac dong lon.

## 6. Postmortem G2 Naive

Tom tat: G2 bat dau khi `alpha` mat 75% nang luc luc t=120. Naive client tao retry storm: availability chi 0.4560, amplification 9.904, 34475 attempts cho 3481 logical calls. Sau khi provider hoi phuc o t=300, recovery van 155s vi hang doi bi bom boi retry cu.

Anh huong: 2362 turns co 1285 turns that bai; p95 thanh cong 15.034s; loi nay tieu khoang 54.4% request trong bang, lon hon nhieu so voi SLO 99.5% neu keo dai theo thang.

| t | Su kien | Bang chung |
|---:|---|---|
| 119 | Truoc su co | 7/7 calls ok, 9 attempts alpha/s |
| 120 | alpha mat 75% nang luc | 7/7 calls ok, 5 attempts alpha/s |
| 150 | Retry storm hinh thanh | 0/3 calls ok, 163 attempts alpha/s |
| 240 | He thong o trang thai metastable | 0/2 calls ok, 105 attempts alpha/s |
| 300 | alpha hoi phuc | 0/5 calls ok, 115 attempts alpha/s |
| 360 | Van chua hoi phuc | 0/1 calls ok, 50 attempts alpha/s |
| 455 | Gan hoi phuc | 4/5 calls ok, 13 attempts alpha/s |

Trigger la mat GPU tren alpha; root cause la 3 tang retry khong jitter/budget/breaker lam tang tai, day hang doi bang request ma client da bo cuoc, roi tiep tuc retry len provider vua hoi phuc. Dieu giup la provider cuoi cung hoi phuc; dieu lam te hon la khong co failover, khong cooldown retry-after, va timeout/retry chong nhau.

| # | Hanh dong | Loai | Owner | Cach kiem chung |
|---|---|---|---|---|
| 1 | Bat retry budget 10% va full jitter | prevent | Platform | G2 amplification <= 1.2 trong CI |
| 2 | Sliding-window breaker co slow-call detection | mitigate | Runtime | G4 p95 <= 10s, G2 recovery <= 30s |
| 3 | Alert amplification/retry-after violations | detect | SRE | Page khi amplification > 2 trong 5m |

## 7. Diem Yeu Con Lai

G2 cua client moi van co p95 10.920s, sat nguong trai nghiem duoi 30s. Lan tiep theo nen them adaptive concurrency hoac queue-depth signal de giam timeout tren alpha som hon, dong thoi uu tien alpha-eu/beta theo latency gan nhat thay vi chi dung thu tu co dinh.

## 8. Cache

| Client | availability | faq_availability | cache_hit_ratio | stale_served | wrong_answers | outdated_answers | cost/turn | PASS? |
|---|---:|---:|---:|---:|---:|---:|---:|:---:|
| resilient | 0.8373 | 0.8424 | 0.000 | 0 | 0 | 0 | 1.386 | no |
| cached | 0.9124 | 0.9645 | 0.652 | 157 | 0 | 3 | 0.915 | yes |

| Tham so `cache:` | Gia tri | Ly do |
|---|---:|---|
| similarity_threshold | 0.95 | 0.80 cho hit 0.742 nhung wrong_answers 181; 0.95 dat wrong_answers 0 va hit 0.652. |
| ttl_s | 60 | Giam outdated_answers con 3 sau thay doi khong co webhook. |
| max_stale_s | 360 | Du phu outage t=240-360; stale_served 157 va faq_availability 0.9645. |

Nguong 0.80 tra nham cau hoi khac (`wrong_answers_by_reason={'other_question': 181}`), nen 0.95 la diem can bang: cost/turn 0.915, wrong_answers 0. TTL 60s khong ngan duoc moi thay doi bang webhook, nhung gioi han thay doi khong bao truoc nen outdated_answers chi 3; TTL 300/600 se giu dap an cu lau hon. Khi `max_stale_s: 0`, FAQ khong co stale-if-error nen gan voi resilient: faq_availability 0.8424 thay vi 0.9645.

Cau hoi ca nhan khong cache ke ca theo tung user vi noi dung phu thuoc trang thai song cua don hang/diem/refund va co rui ro lo du lieu neu scope sai. Khi dashboard van xanh nho cache, can alert them vao provider availability, `stale_served`, cache stale ratio va give_up_reasons de on-call thay provider dang sap du user FAQ van duoc phuc vu.

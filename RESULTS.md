# K6 lab results

## Part 0
- Step 0.1, http_req_duration: avg=min=med=max=p(90)=p(95)=252.79 ms (1 request)
- Step 0.2, requests, rate and p(95), and why the rate stays below 5: 80 requests, 3.86 req/s, p(95) = 260.9 ms. Each iteration is one request (~0.25 s) plus sleep(1), so about 1.28 s per iteration. One user sends about 1/1.28 = 0.78 req/s, and 5 users send about 3.9 req/s, below 5, because every user spends most of each iteration sleeping (think time) and waiting for the response instead of sending requests back to back.
- Step 0.6, exit code of the normal run and of the run with P95=300: Normal run (P95=1500): all thresholds passed, p(95) = 372.41 ms, exit code 0. Run with P95=300: the p(95)<300 rule failed with p(95) = 370.59 ms, K6 reported that the thresholds on http_req_duration were crossed, exit code 99. A CI pipeline reads this non-zero exit code and fails the build, so a slowdown is caught like a failing unit test.
- Step 0.7, rules that passed, p(95), p(99), exit code after your change: With 3 VUs for 30 s, all three rules passed: p(95)<1000 (measured p(95) = 382.95 ms), p(99)<1500 (measured p(99) = 399.92 ms) and http_req_failed rate<0.01 (0.00%). Exit code 0. After changing the first rule to p(95)<200, that rule failed (p(95) = 369.07 ms), the other two still passed, K6 reported that the thresholds on http_req_duration were crossed, and the exit code was 99.
- Step 0.8, what was wrong, and how the checks showed it:

## Part A results
| Step | Test type | Users | Req/s | p(95) | Max | Failed | Checks |
| ---- | ------------ | ----- | ----- | ----- | --- | ------ | ------ |
| A1 | Smoke | 1 | | | | | |
| A2 | Average load | 3 | | | | | |
| A3 | Stress | 5 | | | | | |
| A4 | Spike | 5 | | | | | |
| A5 | Soak | 2 | | | | | |

## Part B
1.
2.
3.
4.
5.
6.
7.
8.
9.
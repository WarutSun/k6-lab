# K6 lab results

## Part 0
- Step 0.1, http_req_duration: avg=min=med=max=p(90)=p(95)=252.79 ms (1 request)
- Step 0.2, requests, rate and p(95), and why the rate stays below 5: 80 requests, 3.86 req/s, p(95) = 260.9 ms. Each iteration is one request (~0.25 s) plus sleep(1), so about 1.28 s per iteration. One user sends about 1/1.28 = 0.78 req/s, and 5 users send about 3.9 req/s, below 5, because every user spends most of each iteration sleeping (think time) and waiting for the response instead of sending requests back to back.
- Step 0.6, exit code of the normal run and of the run with P95=300: Normal run (P95=1500): all thresholds passed, p(95) = 372.41 ms, exit code 0. Run with P95=300: the p(95)<300 rule failed with p(95) = 370.59 ms, K6 reported that the thresholds on http_req_duration were crossed, exit code 99. A CI pipeline reads this non-zero exit code and fails the build, so a slowdown is caught like a failing unit test.
- Step 0.7, rules that passed, p(95), p(99), exit code after your change: With 3 VUs for 30 s, all three rules passed: p(95)<1000 (measured p(95) = 382.95 ms), p(99)<1500 (measured p(99) = 399.92 ms) and http_req_failed rate<0.01 (0.00%). Exit code 0. After changing the first rule to p(95)<200, that rule failed (p(95) = 369.07 ms), the other two still passed, K6 reported that the thresholds on http_req_duration were crossed, and the exit code was 99.
- Step 0.8, what was wrong, and how the checks showed it: The script sent excludedIngredients: ['pepperoni'] and excludedTools: ['knife'] in lowercase, but QuickPizza names them "Pepperoni" and "Knife" (found with explore.js), and the API matches names case-sensitively. The filter therefore matched nothing, the API ignored it and returned random pizzas with status 200 and no error. In the first run (5 VUs, 20 s, 75 requests) the check "no pepperoni" failed 8 times (89% passed) and "no knife needed" failed 30 times (60% passed). After changing the values to 'Pepperoni' and 'Knife', every check passed. One manual request might not show the bug because the pizza is random, so a single pizza can happen to have no pepperoni and no knife by chance (about 40% of pizzas needed no knife by chance). Under load, many requests expose the failure rate, and checks that inspect the response body catch a wrong answer that a status code check (always 200) would miss.

## Part A results
| Step | Test type | Users | Req/s | p(95) | Max | Failed | Checks |
| ---- | ------------ | ----- | ----- | ----- | --- | ------ | ------ |
| A1 | Smoke | 1 | 0.44 | 332 ms | 372.72 ms | 0.00% | 100.00% (20/20) |
| A2 | Average load | 3 | 1.30 | 368.16 ms | 474.23 ms | 0.00% | 100.00% (316/316) |
| A3 | Stress | 5 | 2.18 | 353.15 ms | 379.44 ms | 0.00% | 100.00% (532/532) |
| A4 | Spike | 5 | 1.05 | 371.25 ms | 393.17 ms | 0.00% | 100.00% (196/196) |
| A5 | Soak | 2 | 0.99 | 351.35 ms | 378.96 ms | 0.00% | 100.00% (880/880) |

## Part B

1. The results table for Steps A1 to A5 is in the "Part A results" section above.

2. In Step 0.2, one iteration is one request (about 0.25 s) plus `sleep(1)`, so it takes about 1.28 s. One virtual user therefore sends about 1 / 1.28 ≈ 0.78 requests per second, and 5 users send about 3.9 requests per second. The measured rate was 3.86 req/s. It stays below 5 because each user spends most of every iteration sleeping (think time) and waiting for the response, instead of sending requests back to back.

3. No, the response time did not grow from the average-load test to the stress test. With 3 users (A2) the p(95) was 368.16 ms and the max was 474.23 ms. With 5 users (A3) the p(95) was 353.15 ms and the max was 379.44 ms. The small differences are most likely normal network and server variation, not an effect of load. In terms of the operational envelope, both 3 and 5 users are far inside the range where QuickPizza answers quickly, so we have not reached the edge of the envelope. Finding where the system starts to slow down would need a breakpoint test, which we must not run against a site we do not own. The request rate did rise from 1.30 to 2.18 req/s, roughly in proportion to the number of users.

4. In the spike test (A4), the slowest request of the whole run was 393.17 ms, and the overall p(95) was 371.25 ms, almost the same as the average-load test (368.16 ms). In the report, when the users jumped from 1 to 5 at about 30 to 35 s, the response time to about 381 ms. It returned to its earlier level at about 0.5 s. So QuickPizza handled a sudden jump from 1 to 5 users without any visible slowdown or errors (0.00% failed).

5. In the soak test (A5), the error rate stayed at 0.00% over 660 requests and all 880 checks passed. In the report, the response time in the first minutes was about 268 ms and in the last minutes about 294 ms. Over 11 minutes the max was only 378.96 ms. A soak test of several hours could reveal a memory leak, where the server slowly uses more and more memory until it becomes slow or crashes. It could also reveal resources that are never released, such as database connections or file handles, which run out after a long time.

6. An average hides the slowest requests, because many fast requests pull it down. The p(95) is the time that 95% of requests beat, so it shows what the slowest 1 in 20 users experience. Requirements and thresholds therefore use percentiles, so that a few slow users are not hidden by many fast ones.

7. Two reasons why the same tests against a system in a lab could give different numbers:
   - Network: our requests travel over the internet from Thailand to a remote server, so the measured time includes network latency and its random variation. A lab system is usually on a local network, so its response times would be lower and more stable.
   - Hardware and other users: QuickPizza is shared by learners around the world, and we do not know its hardware or how much other traffic it handles at the same time. A lab system has known hardware and no outside load, so the results are easier to repeat. The numbers could be better or worse, depending on the system.

8. A stress, spike or breakpoint test with hundreds of users can slow down or crash a site, and this would affect the real users of a site we do not own. In practice it can look like a denial-of-service attack. Without written permission it may also break the terms of use or the law, and the owner would pay for the extra resources. Tests like this are only done on our own system or with written permission.

9. App: a chat app that I use every day.
   - Vague requirement: "Sending a message should be fast and should rarely fail."
   - Testable requirement: With 5 users sending messages at the same time for 1 minute, the send-message API answers 95% of requests within 500 ms and 99% within 1 second, and fewer than 1% of requests fail.
   - K6 thresholds:

   ```js
   export const options = {
     vus: 5,
     duration: '1m',
     thresholds: {
       http_req_duration: ['p(95)<500', 'p(99)<1000'],
       http_req_failed: ['rate<0.01'],
     },
   };
   ```

   The numbers 500 ms and 1 second are example targets that I chose, not measured values. I did not run this test, because I do not own the app.
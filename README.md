### Today I learned (TIL)

**August,2nd**

Use a function that is invoked via a cron (every hour) to output GitHub traffic stats which I then use CloudWatch Log Insights to dashboard and get useful info.

```
# based from the script at https://github.com/AnthonyBloomer/github-traffic-insights
# changed so that it outputs just to use Cloudwatch Insights to track GitHub Traffic

import json
import os
from urllib import request


def record_custom_event(event):
    print(json.dumps(event).encode("utf-8"))


def api_req(request_url):
    token = "token " + os.getenv("GITHUB_TOKEN")
    req = request.Request(request_url, headers={"Accept" :"application/vnd.github.v3+json" , "Authorization": token })
    response = request.urlopen(req)
    return json.load(response)


def lambda_handler(event, context):
    base_url = "https://api.github.com"
    user = os.getenv("GITHUB_USERNAME")
    print("Getting repositories for %s" % user)
    req = api_req("%s/users/%s/repos" % (base_url, user))

    ur = [(r["name"]) for r in req]
    print("Found %s repositories" % len(ur))

    print(" ")

    for repo_name in ur:

        print("Getting referral data for %s" % repo_name)
        referrals = api_req("%s/repos/%s/%s/traffic/popular/referrers" % (base_url, user, repo_name))
        if len(referrals) > 0:
            for ref in referrals:
                referred = {
                    "eventType": "Referral",
                    "repo_name": repo_name,
                    "referrer": ref["referrer"],
                    "count": ref["count"],
                    "uniques": ref["uniques"],
                }
                print(referred)
                record_custom_event(referred)

        print("Getting top referral path data for %s" % repo_name)

        paths = api_req("%s/repos/%s/%s/traffic/popular/paths" % (base_url, user, repo_name))
        if len(paths) > 0:
            for ref in paths:
                paths = {
                    "eventType": "ReferralPath",
                    "repo_name": repo_name,
                    "path": ref["path"],
                    "title": ref["title"],
                    "count": ref["count"],
                    "uniques": ref["uniques"],
                }
                print(paths)
                record_custom_event(paths)

        print("Getting views for %s" % repo_name)
        req = api_req("%s/repos/%s/%s/traffic/views" % (base_url, user, repo_name))
        if req["count"] > 0 or req["uniques"] > 0:
            viewed = {
                "eventType": "View",
                "repo_name": repo_name,
                "count": req["count"],
                "uniques": req["uniques"],
            }
            print(viewed)
            record_custom_event(viewed)

        print("Getting clones for %s" % repo_name)
        req = api_req("%s/repos/%s/%s/traffic/clones" % (base_url, user, repo_name))
        if req["count"] > 0 or req["uniques"] > 0:
            cloned = {
                "eventType": "Clone",
                "repo_name": repo_name,
                "count": req["count"],
                "uniques": req["uniques"],
            }
            record_custom_event(cloned)
        print(" ")
```

Once enabled via an hourly cron, it will generate output in CloudWatch which I can then create a dashboard using a filter like:

*Get daily totals across all repos*
```
fields @timestamp, @message
| filter eventType = 'View'
| stats max(count) as total_user_views by bin(1d) as day
```
*Get total view breakdown by repo*
```
fields @timestamp, @message
| filter eventType='View'
| sort @timestamp, repo_name
| stats max(count) as views by repo_name
```



- 👋 Hi, I’m @venugopaldevarapalli
- 👀 I’m interested in ...
- 🌱 I’m currently learning ...
- 💞️ I’m looking to collaborate on ...
- 📫 How to reach me ...

<!---
venugopaldevarapalli/venugopaldevarapalli is a ✨ special ✨ repository because its `README.md` (this file) appears on your GitHub profile.
You can click the Preview link to take a look at your changes.
--->
#!/usr/bin/env python

import boto3
from datetime import datetime, timedelta
import sys
import argparse

parser = argparse.ArgumentParser('Add your team')
parser.add_argument('-t', '--team', type=str, help='Team name"', required=True)
args = parser.parse_args()

client = boto3.client('resourcegroupstaggingapi')

ret_val = 0
paginator = client.get_paginator('get_resources')
pages = paginator.paginate(
    TagFilters=[
        {
            'Key': 'Environment',
            'Values': [
                'Production'
            ],
            'Key': 'Team',
            'Values': [
                '{}'.format(args.team)
            ]
        }
    ],
    ResourceTypeFilters=[
        'lambda',
    ],
)

for page in pages:
    for obj in page['ResourceTagMappingList']:
      lambda_list = (obj["ResourceARN"].split(':', 7 )[-1])

      num = 0.0
      cloudwatch = boto3.client('cloudwatch')
      met = cloudwatch.get_metric_statistics(Namespace='AWS/Lambda',
      StartTime=datetime.now() - timedelta(minutes=10),
          Dimensions=[
              {
                  'Name': 'FunctionName',
                  'Value': lambda_list
              },
          ],
          EndTime=datetime.now(),
          Period=60,
          MetricName="Errors",
          Statistics=['Sum'])
      if len(met['Datapoints']) == 0:
          True
      else:
          for s in met['Datapoints']:
              num = num + s['Sum']
          if int(num) > 1000:
              ret_val = ret_val + 1
          print("{}: {} Errors".format(lambda_list, num))

if ret_val == 0:
    sys.exit(0)
else:
    print("One of the Lambda function has errors over 1k within the last 10 mins, please escalate to {}".format(args.team))
    sys.exit(2)

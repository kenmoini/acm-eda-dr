# ACM and EDA Glue

- Log in as admin
- Add a subscription
- Automation Execution > Infrastructure > Execution Environments, add one like `quay.io/kenmoini/ee-process-policy-violation:latest`
- Automation Decisions > Decision Environments, add one like `quay.io/kenmoini/de-process-policy-violation:latest`
- Automation Execution > Projects, add one from your repo like `https://github.com/kenmoini/acm-eda-dr`
  - If using an Outbound Proxy, configure that in Settings > Job > Extra Environment Varibles
- Automation Decisions > Projects, add one from your repo like `https://github.com/kenmoini/acm-eda-dr`
  - If using an Outbound Proxy, configure that in the EDA Project settings lol
- Automation Execution > Templates, create a Job Template - maybe start with the Debug Playbook.
- Automation Execution > Templates, create a Workflow Template - add the Job Template(s) to it.
- 
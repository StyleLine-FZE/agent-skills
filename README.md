# StyleLine agent skills

Skills that tell an agent how to call a StyleLine product's public customer API. Each skill matches the route handlers in the StyleLine SaaS apps. Install one skill from this repo:

```bash
npx skills add StyleLine-FZE/agent-skills --skill sslclock
npx skills add StyleLine-FZE/agent-skills --skill outageclock
npx skills add StyleLine-FZE/agent-skills --skill pingclock
npx skills add StyleLine-FZE/agent-skills --skill outagepage
npx skills add StyleLine-FZE/agent-skills --skill renewclock
npx skills add StyleLine-FZE/agent-skills --skill firecertclock
npx skills add StyleLine-FZE/agent-skills --skill coiclock
npx skills add StyleLine-FZE/agent-skills --skill cardexpiryclock
npx skills add StyleLine-FZE/agent-skills --skill merchantclock
npx skills add StyleLine-FZE/agent-skills --skill vitalsclock
npx skills add StyleLine-FZE/agent-skills --skill mentionclock
npx skills add StyleLine-FZE/agent-skills --skill teamcakeday
```

| Skill | Product | Public docs |
| --- | --- | --- |
| `sslclock` | SSL Clock | https://www.sslclock.com/docs/api |
| `outageclock` | Outage Clock | https://www.outageclock.com/docs/api |
| `pingclock` | Ping Clock | https://www.pingclock.com/docs/api |
| `outagepage` | Outage Page | https://www.outagepage.com/docs/api |
| `renewclock` | Renew Clock | https://www.renewclock.com/docs/api |
| `firecertclock` | FireCert Clock | https://www.firecertclock.com/docs/api |
| `coiclock` | COI Clock | https://www.coiclock.com/docs/api |
| `cardexpiryclock` | Card Expiry Clock | https://www.cardexpiryclock.com/docs/api |
| `merchantclock` | Merchant Clock | https://www.merchantclock.com/docs/api |
| `vitalsclock` | Vitals Clock | https://www.vitalsclock.com/docs/api |
| `mentionclock` | Mention Clock | https://www.mentionclock.com/docs/api |
| `teamcakeday` | Team Cake Day | https://www.teamcakeday.com/docs/api |

Keys are created in the signed-in dashboard at `/app/settings`. The production base URL for each app is the `productionUrl` in that app's business config.

# ➖ Firecrawl Simple

## ![](https://trieve.b-cdn.net/firecrawl-simple/loc_chart.png)

<div>
  <p align="center"><a href="https://github.com/mendableai/firecrawl/blob/main/LICENSE"><img src="https://img.shields.io/github/license/mendableai/firecrawl" alt="License"></a>
    <a href="https://x.com/trieveai"><img src="https://img.shields.io/badge/Follow%20on%20X-000000?style=flat&amp;logo=x&amp;logoColor=white" alt="Follow on X"></a>
    <a href="https://www.linkedin.com/company/90198314"><img src="https://img.shields.io/badge/Follow%20on%20LinkedIn-0077B5?style=flat&amp;logo=linkedin&amp;logoColor=white" alt="Follow on LinkedIn"></a>
    <a href="https://discord.gg/CuJVfgZf54"><img src="https://img.shields.io/discord/1130153053056684123.svg?style=flat&amp;logo=discord&amp;logoColor=white" alt="Join our Discord"></a></p>
</div>

## What is Firecrawl Simple?

Firecrawl Simple is a stripped down and stable version of firecrawl optimized for self-hosting and ease of contribution. Billing logic and AI features are completely removed.

`playwright` is replaced with `puppeteer-cluster` and `puppeteer-extra`'s stealth plugins such that `fire-engine` and `scrapingbee` are not required for guarded pages. Further, a [2captcha](https://2captcha.com/) token and proxy credentials may be included as ENV's for maximum stealthiness. In the near-term future, the goal is to switch over to [hero](https://github.com/ulixee/hero) since puppeteer-extra is no longer maintained

Only the v1 `/scrape`, `/crawl/{id}`, and `/crawl` routes are supported in firecrawl simple, see the [openapi spec here](/apps/api/v1-openapi.json). Also, `creditsUsed` has been removed from the API response on the `/crawl/{id}` route.

Posthog, supabase, stripe, langchain, logsnag, sentry, bullboard, and [several other deps from the package.json](https://github.com/mendableai/firecrawl/compare/main...devflowinc:firecrawl-simple:main#diff-2c40985d6d91eed8ae85ec1c8e754a85984ee32e156a600d2b7a467423d7e338) are removed.

## Contributing

This is a lot to maintain by ourselves and we are actively looking for others who would like to help. **There are paid part-time maintainer positions available.** We currently have bounties on a couple of issues, but would like someone interested in being an active maintainer longer-term.

## Why maintain a fork?

The [upstream firecrawl repo](https://github.com/mendableai/firecrawl) contains the following blurb:

> This repository is in development, and we're still integrating custom modules into the mono repo. It's not fully ready for self-hosted deployment yet, but you can run it locally.

Firecrawl's API surface and general functionality were ideal for our [Trieve sitesearch product](https://trieve.ai/sitesearch), but we needed a version ready for self-hosting that was easy to contribute to and scale on Kubernetes. Therefore, we decided to fork and begin maintaining a stripped down, stable version.

Fire-engine, Firecrawl's solution for anti-bot pages, being closed source is the biggest deal breaker requiring us to maintain this. Further, our purposes not requiring the SaaS and AI dependencies also pushes our use-case far enough away from Firecrawl's current mission that it doesn't seem like merging into the upstream is viable at this time.

## How to self host?

You should add the following services to your docker-compose as follows. We trust that you can configure Kubernetes or other hosting solutions to run these services.

```yaml
name: firecrawl
services:
  # Firecrawl services
  playwright-service:
    image: trieve/puppeteer-service-ts:v0.0.6
    environment:
      - PORT=3000
      - PROXY_SERVER=${PROXY_SERVER}
      - PROXY_USERNAME=${PROXY_USERNAME}
      - PROXY_PASSWORD=${PROXY_PASSWORD}
      - BLOCK_MEDIA=${BLOCK_MEDIA}
      - MAX_CONCURRENCY=${MAX_CONCURRENCY}
      - TWOCAPTCHA_TOKEN=${TWOCAPTCHA_TOKEN}
    networks:
      - backend

  firecrawl-api:
    image: trieve/firecrawl:v0.0.46
    networks:
      - backend
    environment:
      - REDIS_URL=${FIRECRAWL_REDIS_URL:-redis://redis:6379}
      - REDIS_RATE_LIMIT_URL=${FIRECRAWL_REDIS_URL:-redis://redis:6379}
      - PLAYWRIGHT_MICROSERVICE_URL=${PLAYWRIGHT_MICROSERVICE_URL:-http://playwright-service:3000}
      - PORT=${PORT:-3002}
      - NUM_WORKERS_PER_QUEUE=${NUM_WORKERS_PER_QUEUE}
      - BULL_AUTH_KEY=${BULL_AUTH_KEY}
      - TEST_API_KEY=${TEST_API_KEY}
      - HOST=${HOST:-0.0.0.0}
      - SELF_HOSTED_WEBHOOK_URL=${SELF_HOSTED_WEBHOOK_URL}
      - LOGGING_LEVEL=${LOGGING_LEVEL}
    extra_hosts:
      - "host.docker.internal:host-gateway"
    depends_on:
      - playwright-service
    ports:
      - "3002:3002"
    command: ["pnpm", "run", "start:production"]

  firecrawl-worker:
    image: trieve/firecrawl:v0.0.46
    networks:
      - backend
    environment:
      - REDIS_URL=${FIRECRAWL_REDIS_URL:-redis://redis:6379}
      - REDIS_RATE_LIMIT_URL=${FIRECRAWL_REDIS_URL:-redis://redis:6379}
      - PLAYWRIGHT_MICROSERVICE_URL=${PLAYWRIGHT_MICROSERVICE_URL:-http://playwright-service:3000}
      - PORT=${PORT:-3002}
      - NUM_WORKERS_PER_QUEUE=${NUM_WORKERS_PER_QUEUE}
      - BULL_AUTH_KEY=${BULL_AUTH_KEY}
      - TEST_API_KEY=${TEST_API_KEY}
      - SCRAPING_BEE_API_KEY=${SCRAPING_BEE_API_KEY}
      - HOST=${HOST:-0.0.0.0}
      - SELF_HOSTED_WEBHOOK_URL=${SELF_HOSTED_WEBHOOK_URL}
      - LOGGING_LEVEL=${LOGGING_LEVEL}
    extra_hosts:
      - "host.docker.internal:host-gateway"
    depends_on:
      - playwright-service
      - firecrawl-api
    command: ["pnpm", "run", "workers"]

  redis:
    image: redis:alpine
    networks:
      - backend
    command: redis-server --bind 0.0.0.0

networks:
  backend:
    driver: bridge

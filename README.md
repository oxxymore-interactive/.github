name: Discord Notify (Reusable)

on:
  workflow_call:
    inputs:
      title:
        required: false
        type: string
      url:
        required: false
        type: string
      repo:
        required: false
        type: string
      actor:
        required: false
        type: string
      event:
        required: false
        type: string
    secrets:
      DISCORD_WEBHOOK:
        required: true

permissions:
  contents: read

jobs:
  notify:
    runs-on: ubuntu-latest
    steps:
      - name: Send message to Discord
        env:
          WEBHOOK: ${{ secrets.DISCORD_WEBHOOK }}
          TITLE: ${{ inputs.title }}
          URL: ${{ inputs.url }}
          REPO: ${{ inputs.repo }}
          ACTOR: ${{ inputs.actor }}
          EVENT: ${{ inputs.event }}
          RUN_URL: ${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}
        run: |
          set -euo pipefail

          # Fallbacks (au cas où un input n'est pas transmis)
          TITLE="${TITLE:-GitHub event}"
          URL="${URL:-$RUN_URL}"
          REPO="${REPO:-${GITHUB_REPOSITORY}}"
          ACTOR="${ACTOR:-${GITHUB_ACTOR}}"
          EVENT="${EVENT:-${GITHUB_EVENT_NAME}}"

          payload=$(cat <<EOF
          {
            "content": "**${EVENT}** — ${REPO}\n**${TITLE}**\nBy: ${ACTOR}\n${URL}"
          }
          EOF
          )

          http_code=$(curl -sS -o /tmp/discord_resp.txt -w "%{http_code}" \
            -H "Content-Type: application/json" \
            -X POST \
            -d "$payload" \
            "$WEBHOOK" || true)

          echo "Discord HTTP status: $http_code"
          if [ "$http_code" -lt 200 ] || [ "$http_code" -ge 300 ]; then
            echo "Discord response:"
            cat /tmp/discord_resp.txt || true
            exit 1
          fi

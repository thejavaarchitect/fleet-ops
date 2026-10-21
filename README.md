# fleet-ops

Operational patterns from running a 40+ site production fleet (WordPress-heavy, multi-host) as a
single operator with AI agents as the ops team. These are the generic, reusable pieces — server
hardening includes, deploy discipline, and the gotchas that cost real outages to learn.

> Portfolio note: this repo contains operational tooling and patterns only. Product code
> (agent platforms, compilers, publishing pipeline) is proprietary and not published.

## 1. Hardening includes (nginx)

Maintained as **global includes** (e.g. `/etc/nginx/vhosts-includes/`) rather than per-vhost blocks,
because control panels (ISPmanager, and friends) regenerate per-vhost configs and silently drop
hand-edits. A rule that must survive the panel lives at a layer the panel doesn't own.

**`00-deny-xmlrpc.conf`** — XML-RPC is the most-abused WordPress surface and almost nobody needs it:

```nginx
location = /xmlrpc.php {
    deny all;
    access_log off;
    log_not_found off;
}
```

**`01-deny-dotfiles.conf`** — no `.git`, `.env`, `.htpasswd` ever served (keep `.well-known`):

```nginx
location ~ /\.(?!well-known/) {
    deny all;
    access_log off;
    log_not_found off;
}
```

**`02-deny-backup-ext.conf`** — the safety net for the rule "backups never live in the webroot":

```nginx
location ~* \.(bak|old|orig|save|swp|sql|tar|gz|zip)$ {
    deny all;
}
```

**`03-no-php-in-uploads.conf`** — uploads are data, never code:

```nginx
location ~* ^/wp-content/uploads/.*\.php$ {
    deny all;
}
```

## 2. Deploy discipline (the checklist)

```
backup (OUTSIDE the webroot, keep ≤2) 
  → stage to a temp path
  → lint with the EXACT runtime binary (e.g. /opt/php84/bin/php -l — not whichever php is in $PATH)
  → atomic swap (mv, not cp-over)
  → chown to the site user
  → live verify (HTTP status + the actual feature)
  → log the change (actor, scope, reason)
```

Two rules that earn their keep:
- **JS gets `node --check` before deploy.** A PHP linter will not save you from a broken inline script.
- **Never `ob_start()` inside an output handler.** A mu-plugin that opens a buffer inside a callback
  that is itself flushing output can fatal the entire site. (Cost: one full outage.)

## 3. Gotchas that cost real time

- **Panel-managed PHP:** sites can run a different PHP than the system CLI. `wp-cli` must be invoked
  with the same binary FPM uses, or you lint/execute against the wrong runtime.
- **Log rotation misdirection:** daily rotation with `olddir` means the "live" access log holds only
  minutes of history right after rotation. The history is in the olddir — read both before
  concluding "no traffic."
- **HPOS WooCommerce:** with high-performance order storage, `wp post list --post_type=shop_order`
  returns zero. The orders are in the HPOS tables. Don't declare a store empty from the wrong table.
- **Cache-layer truth:** Varnish/CDN layers can serve a stale page to your health check. Verify
  state-changing flows server-side with a cookie jar, cache-busted.
- **Shared-host OOM math:** on a shared box, per-worker `memory_limit` × worker count is a promise
  you must be able to keep. A "safe" mitigation that raises memory ceilings can be a worse risk
  than the thing it mitigates.

## 4. Money-path verification

A deploy that touches anything near checkout is verified by exercising the path end to end:

```bash
J=$(mktemp)   # cookie jar — survive the cache layers
curl -s -c "$J" -b "$J" -d "add-to-cart=PRODUCT_ID&quantity=1" https://store.example/product/x/
curl -s -c "$J" -b "$J" https://store.example/checkout/ | grep -E "billing|Place Order"
```

"The page returns 200" is not verification. The cart cookie set, the item present, the payment
fields rendered — that's verification.

## License

MIT — see [LICENSE](LICENSE).

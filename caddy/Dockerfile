# Build xcaddy với plugin
FROM caddy:2-builder AS builder

RUN xcaddy build \
    --with github.com/greenpau/caddy-security@latest \
    --with github.com/mholt/caddy-dynamicdns@latest \
    --with github.com/mholt/caddy-ratelimit@latest \
    --with github.com/corazawaf/coraza-caddy/v2

# Copy binary ra final image
FROM caddy:2-alpine
COPY --from=builder /usr/bin/caddy /usr/bin/caddy

# Copy waf config và caddyfile
COPY waf/ /etc/caddy/waf/
COPY Caddyfile /etc/caddy/Caddyfile

# Set working dir
WORKDIR /etc/caddy
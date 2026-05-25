# =============================================================================
# Estágio 1 — Build do plugin a partir do source
# =============================================================================
FROM kong:3.9 AS builder

USER root

RUN apk add --no-cache git zip

RUN git config --global url.https://github.com/.insteadOf git://github.com/

RUN git clone https://github.com/Platformatory/kong-plugin-jwt-keycloak.git /plugin

WORKDIR /plugin

RUN luarocks make kong-plugin-jwt-keycloak-1.3.0-1.rockspec && \
    luarocks pack kong-plugin-jwt-keycloak 1.3.0-1

# =============================================================================
# Estágio 2 — Imagem final com o plugin instalado
# =============================================================================
FROM kong:3.9

USER root

COPY --from=builder /plugin/kong-plugin-jwt-keycloak-1.3.0-1.all.rock /tmp/

RUN luarocks install /tmp/kong-plugin-jwt-keycloak-1.3.0-1.all.rock

USER kong
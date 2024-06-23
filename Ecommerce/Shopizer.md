# Shopizer

## Installation and basic info

Shopizer has 3 parts: backend, administration UI and ecommerce UI. All of those can be deployed as separate containers.

For the backend:
`podman run -p 8080:8080 shopizerecomm/shopizer:latest`

For the admin UI:
`podman run -e "APP_BASE_URL=http://localhost:8080/api" -p 82:80 shopizerecomm/shopizer-admin`

For the ecommerce:
`podman run -e "APP_MERCHANT=DEFAULT" -e "APP_BASE_URL=http://localhost:8080" -p 80:80 shopizerecomm/shopizer-shop-reactjs`

Default username for the administration page is "admin@shopizer.com" password is "password".


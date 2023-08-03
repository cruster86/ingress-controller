- В деплоймент добавлен инит-контейнер с базами данных GeoLite2-Country.mmdb и GeoLite2-ASN.mmdb
- Базы GeoLite2 монтируются через общий вольюм geoip2-db в под контроллера.
- Базы GeoLite2 обновляются автоматически при сборке образа с помощью скрипта.

- Создать секрет gitlab-deploy для доступа к container registry.

```yaml
kind: Deployment
spec:
  template:
    spec:
      imagePullSecrets: 
      - name: gitlab-deploy
      containers:
        name: controller
        volumeMounts:
        - mountPath: /etc/GeoIP2
          name: geoip2-db
      initContainers:
      - command: ["/bin/sh", "-c", "cp -a /opt/* /etc/GeoIP2"]
        image: registry.sirius.online/cheops-edu/geoip2-db:latest
        imagePullPolicy: Always
        name: geoip2-db
        volumeMounts:
        - mountPath: /etc/GeoIP2
          name: geoip2-db
```
---

- Настройка `externalTrafficPolicy: Local` позволяет передавать клиентский ip-адрес к хостинг-бэкенду.

```yaml
kind: Service
spec:
  externalTrafficPolicy: Local
```
---

- Настройка `allow-snippet-annotations: "true"` позволяет добавлять настройки через сниппеты
- Настройка `use-forwarded-headers: "true"` позволяет пробросить заголовки хостинг-бэкенду
- Настройка `main-snippet: load_module` подгружает библиотеку для работы с модулем `geoip2`
- В `http-snippet` определяются стандартные переменные для работы с geoip2 и создаются маппинги в новые переменные `$allowed_country`, `$allowed_asn`, `$grant_access`
  Определяется дополнительные настройки формата лога `log_format custom` с новым полем `$geoip2_data_country_iso_code`
- В `server-snippet` применяется дополнительный формат `custom` для `access_log`

```yaml
kind: ConfigMap
data:
  allow-snippet-annotations: "true"
  http-snippet: |
    geoip2 /etc/GeoIP2/GeoLite2-Country.mmdb {
        $geoip2_data_country_iso_code country iso_code;
        $geoip2_data_continent_code continent code;
    }
    geoip2 /etc/GeoIP2/GeoLite2-ASN.mmdb {
        $geoip2_data_autonomous_system_number autonomous_system_number;
        $geoip2_data_autonomous_system_organization autonomous_system_organization;
    }
    map $geoip2_data_country_iso_code $allowed_country {
        default no;
        AZ yes;
        AM yes;
        BY yes;
        KZ yes;
        KG yes;
        MD yes;
        RU yes;
        TJ yes;
        UZ yes;
    }
    map $geoip2_data_autonomous_system_number $allowed_asn {
        default no;
        6789 yes;
        210078 yes;
        204144 yes;
        204791 yes;
        57679 yes;
        43936 yes;
        47939 yes;
        57847 yes;
        12403 yes;
        57893 yes;
        208397 yes;
        48330 yes;
        44387 yes;
        39248 yes;
        57641 yes;
        203561 yes;
        25591 yes;
        51522 yes;
        197335 yes;
        204161 yes;
        197152 yes;
        197175 yes;
        48722 yes;
        197159 yes;
        203451 yes;
        52052 yes;
        205236 yes;
        48084 yes;
        44533 yes;
        47203 yes;
        196705 yes;
        57093 yes;
        48004 yes;
        197628 yes;
        47349 yes;
        34180 yes;
        51214 yes;
        33978 yes;
        35533 yes;
        209789 yes;
        8654 yes;
        39047 yes;
        207909 yes;
        203454 yes;
        5593 yes;
        47801 yes;
        57903 yes;
        59823 yes;
        211867 yes;
        200702 yes;
        208701 yes;
        209038 yes;
        62287 yes;
        51211 yes;
        210451 yes;
        28917 yes;
        211245 yes;
        443387 yes;
        208090 yes;
        496117 yes;
        51518 yes;
        213258 yes;
        51917 yes;
        56352 yes;
        198338 yes;
        198899 yes;
        44139 yes;
        201069 yes;
        6167 yes;
        212205 yes;
        213278 yes;
        28761 yes;
        35381 yes;
        206448 yes;
        200420 yes;
        207875 yes;
        204259 yes;
        44341 yes;
        52218 yes;
        57982 yes;
        44546 yes;
        210442 yes;
        59744 yes;
        42239 yes;
        49617 yes;
        205515 yes;
        213191 yes;
        209927 yes;
        39529 yes;
        56587 yes;
        211791 yes;
        41161 yes;
        50042 yes;
        200922 yes;
        57874 yes;
        50140 yes;
        43822 yes;
        41269 yes;
        43222 yes;
        43564 yes;
        35816 yes;
        59833 yes;
        201776 yes;
        57383 yes;
        198293 yes;
        43634 yes;
        34901 yes;
    }
    map "$geoip2_data_country_iso_code:$allowed_asn" $grant_access {
        default no;
        "~AZ:.*" yes;
        "~AM:.*" yes;
        "~BY:.*" yes;
        "~KZ:.*" yes;
        "~KG:.*" yes;
        "~MD:.*" yes;
        "~RU:.*" yes;
        "~TJ:.*" yes;
        "~UZ:.*" yes;
        "~UA:yes" yes;
    }
    log_format custom '$remote_addr - $remote_user [$time_local] "$request" $status $body_bytes_sent "$http_referer" "$http_user_agent" $request_length $request_time [$proxy_upstream_name] [$proxy_alternative_upstream_name] $upstream_addr $upstream_response_length $upstream_response_time $upstream_status $req_id $geoip2_data_country_iso_code';
  main-snippet: load_module /etc/nginx/modules/ngx_http_geoip2_module.so;
  server-snippet: |
    access_log /var/log/nginx/access.log custom if=$loggable;
  use-forwarded-headers: "true"
```
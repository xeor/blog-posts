xeor/hass:2020.12.0

      - name: init-secrets
        emptyDir:
          medium: Memory
      - name: init-secrets-ro
        secret:
          secretName: '{{ instance }}--{{ name }}'

      initContainers:
      - name: base-secret-injector
        image: alpine
        args:
        - "/bin/sh"
        - "-c"
        - "cp /etc/init-secrets-ro/* /etc/init-secrets/"
        volumeMounts:
        - mountPath: "/etc/init-secrets"
          name: init-secrets
        - mountPath: "/etc/init-secrets-ro"
          name : init-secrets-ro
          
          
{% if mounts_deletable_secrets %}
apiVersion: v1
kind: Secret
metadata:
  name: '{{ instance }}--{{ name }}'
type: Opaque
stringData:
{% for name, data in mounts_deletable_secrets.items() %}
  {{ name }}: {{ data }}
{% endfor %}
{% endif %}%                      

```
Name:             web-ingress
Labels:           app.kubernetes.io/managed-by=Helm
Namespace:        default
Address:          k8s-default-webingre-6dd1452234-672546948.us-west-2.elb.amazonaws.com
Ingress Class:    alb
Default backend:  <default>
Rules:
  Host        Path  Backends
  ----        ----  --------
  *           
              /   model-serve:9000 (192.168.20.213:80)
Annotations:  alb.ingress.kubernetes.io/scheme: internet-facing
              alb.ingress.kubernetes.io/target-type: ip
              meta.helm.sh/release-name: emlo-s18
              meta.helm.sh/release-namespace: default
Events:
  Type    Reason                  Age   From     Message
  ----    ------                  ----  ----     -------
  Normal  SuccessfullyReconciled  109s  ingress  Successfully reconciled
  ```

1.
Để tìm resource trên toàn bộ các namespace trong Kubernetes, dùng option --all-namespaces (hoặc viết tắt -A) với lệnh kubectl.

Ví dụ:

Liệt kê pods tất cả namespace:

`kubectl get pods --all-namespaces`


kubectl get pods --all-namespaces | grep ContainerStatusUnknown | awk '{print "kubectl delete pod " $2 " -n " $1}' | bash



kubectl run curl-test --image=curlimages/curl --rm -it --restart=Never -- curl http://<ten-service>:<port>t>

k run --image=curlimages/curl mycurl -- sh -c 'sleep infinity'


kubectl patch svc $servicename --type='json' -p '[{"op":"replace","path":"/spec/type","value":"NodePort"}]'; kubectl get svc $servicename -o jsonpath='{.spec.ports[0].nodePort}'; echo

Lệnh này dùng để:

Chuyển kiểu Service sang NodePort

Lấy nodePort của port đầu tiên trong Service đó

In ra nodePort ra terminal



"C:\Program Files\Google\Chrome\Application\chrome.exe" --disable-web-security --user-data-dir="C:/ChromeDevSession"

3. k logs -f --tail=100

#### kubectl get events -n argocd --sort-by='.lastTimestamp'
xem log

4. kubectl -n argocd rollout restart deployment argocd-image-updater
Restart pod

5. Trong mỗi container/pod đều có các biến môi trường lưu thông tin về cụm K8s (VD host, port,...), in ra bằng lệnh env

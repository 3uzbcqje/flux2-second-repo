export GITHUB_TOKEN=REDACTED
export GITHUB_USER=evanstucker-hates-2fa
export GITHUB_REPO=flux2-second-repo

flux create secret git flux2-second-repo \
  --url=ssh://git@github.com/evanstucker-hates-2fa/flux2-second-repo \
  --ssh-key-algorithm=ecdsa \
  --ssh-ecdsa-curve=p521

kubectl get secret -n flux-system flux2-second-repo -o go-template='{{range $k,$v := .data}}{{printf "%s: " $k}}{{if not $v}}{{$v}}{{else}}{{$v | base64decode}}{{end}}{{"\n"}}{{end}}'

Copy the identity.pub, and add it as a Deploy Key to the flux2-second-repo GitHub repo. 

kubectl apply -f clusters/test/flux-system/gotk-sync.yaml

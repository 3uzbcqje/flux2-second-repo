export GITHUB_TOKEN=REDACTED
export GITHUB_USER=evanstucker-hates-2fa
export GITHUB_REPO=flux2-second-repo
flux bootstrap github \
    --context=kind-test \
    --owner=${GITHUB_USER} \
    --repository=${GITHUB_REPO} \
    --branch=main \
    --personal \
    --path=clusters/test

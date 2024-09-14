# Nvidia-Actions-Runner (and Dind) for ARC controller on Kubernetes

**if you want to run a self-hosted github runners with GPU support on kubernetes, these images may help you.**


CUDA 12 and Ubuntu 22.04 based Actions Runner image and Nvidia-DinD (Docker in Docker) image, optimized for use with https://github.com/actions/actions-runner-controller

## Usage With helm

```bash
helm upgrade -i "${RELEASE_NAME}" -f template.yaml \
    --namespace "${NAMESPACE}" \
    --create-namespace \
    --set githubConfigUrl="${GITHUB_CONFIG_URL}" \
    --set githubConfigSecret.github_token="${GITHUB_PAT}" \
    oci://ghcr.io/actions/actions-runner-controller-charts/gha-runner-scale-set
```



### Nvidia Actions Runner Template
```yaml
template:
  spec:
    tolerations:
      - key: "nvidia.com/gpu"
        operator: "Exists"
        effect: "NoSchedule"
    containers:
      - name: runner
        image: isnob46/nvidia-actions-runner:cuda12-2.319
        command:
          - /home/runner/run.sh
```

### Nvidia-DinD-Runner Template  

```yaml
template:
spec:
    tolerations:
    - key: "nvidia.com/gpu"
        operator: "Exists"
        effect: "NoSchedule"
    initContainers:
    - name: init-dind-externals
        image: isnob46/nvidia-actions-runner:cuda12-2.319
        command:
        ["cp", "-r", "-v", "/home/runner/externals/.", "/home/runner/tmpDir/"]
        volumeMounts:
        - name: dind-externals
            mountPath: /home/runner/tmpDir
    containers:
    - name: runner
        image: isnob46/nvidia-actions-runner:cuda12-2.319
        command: ["/home/runner/run.sh"]
        env:
        - name: DOCKER_HOST
            value: unix:///run/docker/docker.sock
        volumeMounts:
        - name: work
            mountPath: /home/runner/_work
        - name: dind-sock
            mountPath: /run/docker
            readOnly: true
    - name: dind
        image: isnob46/nvidia-dind:cuda12
        args:
        - dockerd
        - --host=unix:///run/docker/docker.sock
        - --group=$(DOCKER_GROUP_GID)
        env:
        - name: DOCKER_GROUP_GID
            value: "123"
        securityContext:
        privileged: true
        volumeMounts:
        - name: work
            mountPath: /home/runner/_work
        - name: dind-sock
            mountPath: /run/docker
        - name: dind-externals
            mountPath: /home/runner/externals
    volumes:
    - name: work
        emptyDir: {}
    - name: dind-sock
        emptyDir: {}
    - name: dind-externals
        emptyDir: {}
```


# Acknowledgement

- https://github.com/Extrality/nvidia-dind
- https://github.com/actions/runner
- https://github.com/actions/actions-runner-controller
- https://github.com/actions/actions-runner-controller/discussions/3409
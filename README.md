<div style="display: flex; align-items: center; gap: 40px;">

  <!-- Logo po lewej -->
  <img src="https://raw.githubusercontent.com/AgataGolik/images/main/xCEJIOue_400x400.jpg"
       alt="Logo" width="260">

  </pre>

</div>

<div align="center">

<h1 style="color:white; background-color:#ff5555; padding:30px; border-radius:24px;">
Check Your CPU on VAST Before Running CodeAssist
</h1>

</div>


## Why this matters

When you rent a server on VAST, it's easy to end up with a machine that looks great on paper but performs poorly in practice. Before installing CodeAssist or any heavy AI tooling, you should verify CPU performance, network speed, and the overall stability of the instance. This guide walks you through the process.

---

## We create a screen session so we always have it at hand and can jump back to it anytime:

```
screen -S test
```


## 1. Identify your CPU

After logging into your server via SSH, start by checking the CPU model:

```bash
lscpu | grep -E "Model name|CPU MHz|Core|Socket"
```
![images](https://github.com/AgataGolik/images/blob/main/test0.jpg)

### Example result (AMD Ryzen 9 9950X):

```
Model name:                           AMD Ryzen 9 9950X 16-Core Processor
Core(s) per socket:                   1
Socket(s):                            30
```

This means the machine provides 30 virtual CPUs (30 vCPUs).



---

## 2. Check average CPU clock speed

This helps you see if the processor is artificially throttled.

```bash
cat /proc/cpuinfo | awk -F: '/^cpu MHz/ {sum+=$2; n++} END {if(n) printf "Average CPU MHz: %.2f\n", sum/n; else print "no data"}'

```

![images](https://github.com/AgataGolik/images/blob/main/test1.jpg)

### Example result for me:

```
Average CPU MHz: 4291.95
```

If you see numbers far below 2000 MHz, the CPU may be limited.

---

## 3. Sysbench CPU benchmark

One of the most common ways to measure CPU speed.

### Install sysbench:

```bash
apt update && apt install sysbench -y
```

### Run the test:

```bash
sysbench cpu --cpu-max-prime=20000 run
```
![images](https://github.com/AgataGolik/images/blob/main/test2.jpg)

### Example result:

```
CPU speed:

events per second:  2615 (26155/10 total time)
latency avg: 0.38 ms

That’s a killer result for sysbench with cpu-max-prime=20000.

For context:
* weak or heavily capped CPUs → 700–1200 events/s
* mid-range Ryzen 5/7 → 1500–2000
* Ryzen 9 7950X / 9950X with no throttling → 2300–2700

Hitting 2615 here puts this instance in the genuinely high-performance tier.

```

A result of **2000–3000+ events/s** is considered very good.

---

## 4. Stress-ng stability test

This checks stability under load for 60 seconds.

### Install:

```bash
apt install stress-ng -y
```

### Run the test:


```bash
stress-ng --cpu 30 --cpu-method matrixprod --timeout 60s --metrics-brief
```
You must wait here for 60 seconds.

![images](https://github.com/AgataGolik/images/blob/main/test3.jpg)

### Example output:

```
stressor cpu: 255473 bogo ops in 60 seconds
bogo ops/s: 4257.65
```

Values above ~3500 bogo ops/s are very good.

---

## 5. Additional CPU test (mpstat)

If you want to check the load on each individual core, you can use mpstat:

```
apt install sysstat -y
```
```
mpstat -P ALL 1 5
```

This will show per-core CPU usage sampled every 1 second, for a total of 5 intervals.

![images](https://github.com/AgataGolik/images/blob/main/test4.jpg)

If you see something like this showing 100%, it basically means that your instance:

isn't oversold,
has the full 30 sockets (which in practice means 30 cores),
shows zero steal time,
and runs perfectly stable.

---

## 6. Test your internet speed

Network performance also matters for CodeAssist.

### Quick ping test:

```bash
ping -c 4 google.com
```
and
```bash
mtr -rw google.com
```
![images](https://github.com/AgataGolik/images/blob/main/test6.jpg)

This CPU has got:

average ping: 5.8 ms
packet loss: 0%
stability: mdev 0.338 ms (very little fluctuation)

For comparison:

typical VPS servers sit around 20–40 ms,
weaker or cheaper machines can even hit 50–120 ms,
5–10 ms is the premium data center class, usually with direct peering to Google.

MTR: 

Wrst (worst) 6.7 ms – no spikes at all -  that’s insanely stable.

On weaker machines you often see:
jumps to 60–100 ms,
sometimes even 200 ms,
packet loss.

### Speedtest CLI:

```bash
apt install speedtest-cli -y
```
```bash
speedtest
```

Example:

```
Download: 1903 Mbit/s
Upload:   1395 Mbit/s
```

---

## 7. Check RAM and disk space

### RAM:

```bash
free -h
```

Example:

```
Mem: 197Gi available
```

### Disk:

```bash
df -h
```

---

## 8. Red flags to watch for

* very low CPU clock (e.g., 1200 MHz)
* sysbench results below 1000 events/s
* poor upload speed (<200 Mbit/s)
* unstable stress-ng output (errors, throttling)
* RAM below 32 GB (too little for large models)

---

## 9. When you should switch to another VAST instance

Replace the machine if you notice:

* CPU throttling
* unusually high system load with no activity
* very weak sysbench numbers
* slow disk (I/O < 500 MB/s)

If you keep such a server, CodeAssist will perform poorly.

---

## 10. Summary

This guide helps you quickly evaluate a VAST server before installing CodeAssist. It protects you from paying for a machine that looks good but performs badly.

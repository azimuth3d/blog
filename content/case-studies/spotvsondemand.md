+++
date = '2024-07-04T21:32:12+07:00'
draft = false
title = 'จ่ายแพงกว่าทำไม? เจาะลึกวิธีรัน Production บน K8s ด้วย Spot + On-Demand'
language = 'th'
+++

![Cloud Instance ](/images/spotvsondemand.png)

### Cloud Instance มีประเภทไหนบ้าง 

Cloud Instance ภายใน Cloud Provider ชั้นนำอย่าง AWS, GCP หรือ Azure สามารถแบ่งประเภทการใช้งานหลักๆ ได้เป็น 2 รูปแบบ เพื่อตอบโจทย์ทั้งด้านความเสถียรและต้นทุน:

- On-Demand Instance: เป็นรูปแบบการใช้งานมาตรฐานที่เราเช่าใช้แบบรายชั่วโมงหรือรายวินาที ให้ความมั่นใจในด้าน Availability สูงสุด (24/7) เหมาะสำหรับ Workload ที่ต้องการความเสถียรและรันต่อเนื่องตลอดเวลา

- Spot Instance: ทางเลือกสำหรับสาย Optimized Cost ที่สามารถลดค่าใช้จ่ายได้ถึง 60 - 80% จากราคา On-Demand! แต่มาพร้อมกับข้อตกลงสำคัญคือ Cloud Provider มีสิทธิ์ 'ยึดคืน' (Preempt) Instance ของเราได้ตลอดเวลา หากระบบต้องการ Capacity ไปใช้ในส่วนอื่น ซึ่งหมายความว่าแอปพลิเคชันของเราต้องถูกออกแบบมาให้รองรับการหยุดทำงานหรือการย้าย Node ได้โดยไม่กระทบต่อภาพรวมระบบ


เราควรจัดการ Workload แบบไหนดีให้เหมาะสมที่สุด ?

### The Golden Ratio: เล่าถึงสถาปัตยกรรมที่แบ่ง Node Pool เป็น 2 ส่วน:

- On-Demand Pool: สำหรับ Core Services ที่ห้ามตายเด็ดขาด (เช่น APISIX Ingress, Database, StatefulSet, ETCD)
- Spot Pool: สำหรับ Stateless Workloads(เช่น API Workers, AI Inference API, Background Jobs)

### Prerequisites: สิ่งที่ต้องเตรียมตัวก่อนย้าย Workload ลง Spot

Stateless by Design: แอปพลิเคชันต้องไม่เก็บ State ไว้ใน Memory หรือ Local Disk หากถูกเตะออกกลางคัน State ต้องยังอยู่ใน Database (เช่น ScyllaDB หรือ Redis)

Pod Disruption Budgets (PDB): การบังคับให้ K8s รู้ว่า "จะเตะ Pod ฉันออกไม่ว่า แต่ต้องเหลือ Pod รันอยู่อย่างน้อย X ตัวเสมอ"

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: api-worker-pdb
spec:
  minAvailable: 50%
  selector:
    matchLabels:
      app: api-worker
```

### Deep Dive: ศิลปะแห่งการตายอย่างสงบ (Graceful Shutdown)
เมื่อ Node กำลังจะถูกดึงคืน Cloud Provider มักจะส่งสัญญาณเตือน (Termination Notice) มาล่วงหน้าประมาณ 25-30 วินาที K8s จะจับสัญญาณนี้และเริ่มกระบวนการ Evict Pod

การจัดการ SIGTERM: แอปพลิเคชันต้องถูกเขียนมาให้ดักจับ (Catch) สัญญาณ SIGTERM เมื่อได้รับปุ๊บ ต้องเลิกรับ Request ใหม่ทันที แต่ให้เวลาประมวลผล Request เก่าที่ค้างอยู่ให้เสร็จ แล้วค่อยคืน Memory (Exit 0)

preStop Hook (The Safety Net): โชว์วิธียื้อเวลาและถอดตัวเองออกจาก Load Balancer หรือ Service Discovery เพื่อไม่ให้ Traffic ใหม่วิ่งเข้ามาหา Pod ที่กำลังจะตาย

```yaml
lifecycle:
  preStop:
    exec:
      command: ["/bin/sh", "-c", "sleep 10 && nginx -s quit"]
```

การ sleep 10 ช่วยให้ Kube-proxy อัปเดต iptables และตัด IP ของ Pod นี้ออกจาก Endpoint ล่วงหน้า ป้องกันปัญหา 502 Bad Gateway

### Zero-Downtime ด้วย Health Checks ที่แม่นยำ
เมื่อ Pod บน Spot Node ตาย K8s จะต้องรีบสร้าง Pod ใหม่ขึ้นมาทดแทน (หรือย้ายไป On-Demand ถ้า Spot หมด)
- Readiness Probe: ต้องตั้งให้เป๊ะ เพื่อให้มั่นใจว่า Pod ใหม่พร้อมรับ Traffic จริงๆ ไม่ใช่แค่ Start Process ติด
- Liveness Probe: อย่าตั้งให้ Sensitive เกินไปจน Pod รีสตาร์ทตัวเองรัวๆ จังหวะที่โหลดหนัก

### Node Affinity & Tolerations (การกำหนดว่า Workload ควรจะทำงานอยู่บนไหนบ้าง)
เราจะใช้ Node Affinity  และ toleations เพื่อบังคับให้ Pod เลือกลง Spot Node ก่อน ถ้า Spot ไม่มี ค่อยไปลง On-Demand (Fallback)

```yaml
affinity:
  nodeAffinity:
    preferredDuringSchedulingIgnoredDuringExecution:
    - weight: 100
      preference:
        matchExpressions:
        - key: cloud.google.com/gke-spot   ### สำหรับ  GCP  
          operator: In
          values:
          - "true"

```

### บทสรุป: Trade-off ที่คุ้มค่า
จากประสบการณ์ ปรับการใช้  Node pool ให้เหมาะสมกับ Workload นั้น สามารถลดค่าใช้จ่ายได้สูงมาก โดยเฉพาะระบบที่้มี Worker เป็น Stateless เป็นส่วนใหญ่ พวก concurrent job ต่างๆ 

"Infrastructure ที่ดี ไม่ใช่แค่การเขียนโค้ดเก่ง แต่คือการเข้าใจ Lifecycle ของระบบและจัดการความเสี่ยงได้อย่างสมบูรณ์แบบ"

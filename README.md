# AIS K8s Pre POC Lab Config

ดู [คู่มือ Deploy และลำดับการติดตั้ง](INSTALL.md) สำหรับ requirements และขั้นตอน deploy YAML จนถึงการทดสอบบริการ


## Pre-requisites

- Kubernetes Cluster
- NFS Storage

## การตั้งชื่อ deployment

ใช้ namespace `smlerp` สำหรับ ERP, Stock Process, pgAdmin และตัวอย่างทดสอบ NFS โดยใช้ `smlerp` แทน `imexerp`/`imex-erp` ในชื่อ resource, labels, selectors, จุดอ้างอิง และชื่อไฟล์/โฟลเดอร์ที่เกี่ยวข้อง ส่วน NFS provisioner คงการตั้งค่าเดิม

การปรับชื่อนี้ครอบคลุมไฟล์และเอกสารใน repository เท่านั้น การย้าย workload หรือข้อมูลจาก namespace เดิมบน cluster ต้องวางแผนแยกต่างหาก

<!-- cSpell:disable -->
<!-- markdownlint-disable -->
# 📖 คู่มือการใช้งาน The AI-SOAR Analyst
**เวอร์ชัน:** 1.0 (n8n + Local LLM Orchestration)

เอกสารฉบับนี้คือคู่มือสำหรับผู้ดูแลระบบ (System Administrator) และนักวิเคราะห์ความปลอดภัย (SOC Analyst) เพื่อตั้งค่าและใช้งานระบบ The AI-SOAR Analyst อย่างเต็มรูปแบบ

---

## 🏗️ 1. ภาพรวมระบบ (System Overview)
ระบบนี้ถูกออกแบบมาเพื่อลดภาระของ SOC Analyst โดยใช้ AI มาช่วยอ่านและคัดกรอง Log อัตโนมัติ:
1. **Wazuh** แจ้งเตือนเมื่อพบภัยคุกคาม
2. **n8n** รับเรื่องและส่งข้อมูลให้ **Ollama (Llama 3.2)** วิเคราะห์
3. AI สรุปเนื้อหาและประเมินความเสี่ยงส่งเข้า **Telegram**
4. Analyst กดปุ่มตอบสนอง (เช่น Block IP) ผ่านแชท
5. n8n นำคำสั่งไปยิง API สั่งให้ Wazuh แบนผู้บุกรุกทันที

---

## 🚀 2. การเปิดใช้งานระบบพื้นฐาน (Startup)
1. เปิด Terminal / PowerShell ไปที่โฟลเดอร์โปรเจกต์ `SOC_The AI_SOAR Analyst`
2. รันคำสั่งเปิดใช้งานเซิร์ฟเวอร์ n8n และ AI:
   ```powershell
   docker-compose up -d
   ```
3. **สำคัญมาก:** ต้องสั่งให้ระบบดาวน์โหลดโมเดลสมองกล (Llama 3.2) ลงมาที่เครื่องก่อน (ทำแค่ครั้งแรกครั้งเดียว):
   ```powershell
   docker exec -it ai_soar_ollama ollama pull llama3.2
   ```

---

## ⚙️ 3. การตั้งค่าระบบอัตโนมัติ (n8n Configuration)
1. เปิด Web Browser ไปที่ `http://localhost:5678`
2. สร้างบัญชีผู้ใช้งาน (Owner Account)
3. ไปที่เมนู **Workflows** ด้านซ้ายมือ ➡️ กดปุ่ม **Add Workflow**
4. คลิกไอคอน **จุด 3 จุด (⋯)** มุมขวาบน ➡️ เลือก **Import from File**
5. เลือกไฟล์ `n8n/workflow_ai_soar.json` 
6. **ตั้งค่าจุดเชื่อมต่อ (Credentials) 3 จุดดังนี้:**
   * **จุดที่ 1:** ดับเบิลคลิก Node `Get Wazuh Token` 
     - กดสร้าง Credential ใหม่ (HTTP Basic Auth)
     - ใส่ Username / Password ของ **Wazuh Manager** ของคุณ
     - กด Save
   * **จุดที่ 2:** ดับเบิลคลิก Node `Telegram (Human-in-the-loop)`
     - กดสร้าง Credential ใหม่ 
     - ใส่ **Bot Token** ที่ได้จาก `@BotFather`
     - ใส่ **Chat ID** ของคุณ (หาได้จาก `@userinfobot`)
   * **จุดที่ 3:** ดับเบิลคลิก Node `Execute Block IP (Wazuh AR)`
     - เลื่อนลงมาที่ช่อง URL เปลี่ยนคำว่า `wazuh-manager` เป็น **IP ของเครื่อง Wazuh ของคุณ** (เช่น `https://192.168.1.100:55000/active-response`)
7. กดสวิตช์ที่มุมขวาบนให้กลายเป็น **Active**

---

## 🛡️ 4. การเชื่อมต่อฝั่ง Wazuh Manager
เพื่อให้ Wazuh ส่งข้อมูลมาให้ n8n ได้ คุณต้องไปตั้งค่าที่เครื่อง Wazuh Manager:

1. รีโมท (SSH) เข้าไปที่เครื่อง Wazuh Manager
2. แก้ไขไฟล์คอนฟิก:
   ```bash
   sudo nano /var/ossec/etc/ossec.conf
   ```
3. คัดลอกโค้ดด้านล่างไปวาง **ก่อนแท็ก** `</ossec_config>` บรรทัดสุดท้ายของไฟล์:
   ```xml
   <integration>
     <name>custom-n8n</name>
     <!-- เปลี่ยน [IP_N8N] เป็นไอพีเครื่องที่รัน Docker n8n -->
     <hook_url>http://[IP_N8N]:5678/webhook/wazuh-webhook</hook_url>
     <level>7</level>
     <alert_format>json</alert_format>
   </integration>
   ```
4. รีสตาร์ท Wazuh เพื่อใช้การตั้งค่าใหม่:
   ```bash
   sudo systemctl restart wazuh-manager
   ```

---

## 👨‍💻 5. วิธีการใช้งานจริงสำหรับ SOC Analyst
หลังจากตั้งค่าเสร็จสิ้น ระบบจะทำงานอัตโนมัติตลอด 24 ชม. บทบาทของคุณจะเหลือเพียงแค่:

1. **รอรับการแจ้งเตือน:** เมื่อมี Alert ความรุนแรงสูง (Level 7+) คุณจะได้รับข้อความใน Telegram ทันที
2. **อ่านบทสรุป AI:** AI จะเขียนสรุป (Summary) และบอกว่าเป็น False Positive หรือไม่ พร้อมแนะนำวิธีการรับมือ
3. **ตัดสินใจ (Human-in-the-loop):**
   * หากอันตรายจริง: กดปุ่ม `🚨 Block IP` ระบบจะสั่งบล็อก IP นั้นไม่ให้โจมตีได้อีก
   * หากเป็นการแจ้งเตือนลวง: กดปุ่ม `✅ Ignore` ระบบจะเพิกเฉยและไม่ทำอะไร
4. **รอรับการยืนยัน:** เมื่อกด Block IP สำเร็จ บอทจะตอบกลับมาว่า *"🛡️ The IP has been successfully blocked by Wazuh Active Response."*

---

> [!TIP]
> **การตรวจสอบความผิดปกติ (Troubleshooting):**
> หากตั้งค่าแล้วบอทไม่ยอมส่งข้อความ ให้เข้าไปที่หน้า n8n ➡️ ดับเบิลคลิกที่ Node ขวาสุด ➡️ เลือกแท็บ **Executions** คุณจะเห็นประวัติการทำงานและจุดที่เกิด Error ได้อย่างชัดเจน

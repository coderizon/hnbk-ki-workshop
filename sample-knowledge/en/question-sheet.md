# Question Sheet for Trying the Knowledge Base

Use these questions to test retrieval after you have uploaded the four documents
into Open WebUI. Each question lists the expected answer so you can check whether
the system really answers from the documents.

## How to proceed

1. In Open WebUI, go to **Workspace → Knowledge** and create a collection,
   e.g. "Sample documents".
2. Upload the **four** documents:
   `it-policy.md`, `fire-safety-policy.md`,
   `product-datasheet-packmule-x2.pdf`, `excursion-hannibal-colliery.pdf`.
   (This file and the `README.md` are **not** uploaded.)
3. Start a chat, attach the knowledge collection to your question, and ask the
   questions below.
4. Under each answer, look at the **cited sources**: which document and section did
   the answer come from?

## Questions on the IT policy

1. **What is the teaching Wi-Fi called, and where do you find the password?**
   *Expected:* HNBK-Lab; the password is posted in the school office in Room A101.
2. **How many pages may a student print per term, and where is colour printing possible?**
   *Expected:* 500 pages per term; colour printing only in the library.
3. **What are the two local servers called, and which room are they in?**
   *Expected:* Spark-North and Spark-South, in server room B114.

## Questions on the fire safety policy

1. **Where is the assembly point in the event of a fire?**
   *Expected:* The North car park opposite Hall 3.
2. **Who is the fire safety officer, and where can they be found?**
   *Expected:* Ms. Keller, Room A004 (extension 204).
3. **How often do evacuation drills take place?**
   *Expected:* Twice a year, in autumn and in spring.

## Questions on the product datasheet (PackMule X2 cargo bike)

1. **What is the maximum load, and how far does the battery reach?**
   *Expected:* 180 kg load; range up to 90 km.
2. **What does the PackMule X2 cost, and how long is the warranty on the frame?**
   *Expected:* €4,299; 5-year warranty on the frame (2 years on the battery).
3. **Which colours is the cargo bike available in?**
   *Expected:* Fir Green, Anthracite, and Signal Red.

## Questions on the excursion (Hannibal Colliery)

1. **When and where is the meeting point for the excursion?**
   *Expected:* Friday, 25 September 2026, at the school main entrance at 8:15 AM.
2. **What does the excursion cost, and by when must it be paid?**
   *Expected:* €12 per person; in cash by 18 September 2026 to Ms. Vogt.
3. **What do you need to bring?**
   *Expected:* Sturdy shoes, a rain jacket, your own food, and your student ID. The
   helmet is provided on site.

## Cross-document questions

These can only be answered if the system combines **two** documents.

1. **What is the extension for IT support, and what is the one for the internal security service?**
   *Expected:* IT support 230 (from the IT policy); security service 555 (from the
   fire safety policy).
2. **Which rooms have CO2 fire extinguishers, and what does the IT policy say is in one of those rooms?**
   *Expected:* CO2 extinguishers in B112 and B114; B114 holds the servers
   Spark-North and Spark-South, and B112 is the IT support room.

## Grounding test (answer is NOT in the documents)

These check whether the system stays honest instead of guessing. A good answer
says, in effect, that the information is not in the documents.

1. **What is the current Wi-Fi password for HNBK-Lab?**
   *Expected:* The password is in none of the documents (it is only posted in the
   office). The system should say so and not invent a value.
2. **How many employees does Ruhrrad Manufaktur GmbH have?**
   *Expected:* This detail is not contained in the documents.

## Tip

Ask the same question once **with** and once **without** the knowledge collection
attached. Without the documents the model cannot know the school-specific facts;
with them it can. That is the clearest way to see what RAG adds.

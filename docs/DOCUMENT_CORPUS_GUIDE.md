# ULBS Document Corpus Preparation Guide
## Member 5 Deliverable

---

## 1. Corpus Organization Structure

All university documents must be placed in the Docker-mounted volume at `/data/documents/` with the following structure:

```
/data/documents/
├── academic/
│   ├── calendarul_academic_2025_2026.pdf
│   ├── regulament_studii_licenta.pdf
│   ├── regulament_studii_master.pdf
│   ├── regulament_sustinere_licenta.pdf
│   ├── regulament_sustinere_disertatie.pdf
│   ├── plan_invatamant_informatica.pdf
│   ├── plan_invatamant_litere.pdf
│   ├── ghid_student_anul_I.pdf
│   └── programa_analitica/
│       ├── informatica_sem1.pdf
│       └── informatica_sem2.pdf
│
├── administrative/
│   ├── taxe_scolarizare_2025_2026.pdf
│   ├── regulament_burse.pdf
│   ├── regulament_cazare_camine.pdf
│   ├── ghid_inscriere_admitere.pdf
│   ├── formulare/
│   │   ├── cerere_bursa.docx
│   │   ├── cerere_certificat_student.docx
│   │   └── cerere_echivalare_credite.docx
│   ├── contact_secretariate.docx
│   └── program_functionare_secretariat.pdf
│
├── campus/
│   ├── harta_campus.pdf
│   ├── biblioteca_program.pdf
│   ├── cantina_meniu.pdf
│   └── facilitati_sportive.pdf
│
└── faq/
    ├── intrebari_frecvente_admitere.docx
    ├── intrebari_frecvente_studenti.docx
    └── intrebari_frecvente_erasmus.docx
```

---

## 2. Supported File Formats

| Format | Priority | Notes |
|--------|----------|-------|
| PDF | Primary | Most university docs are PDF. Ensure text is selectable (not scanned images) |
| DOCX | Secondary | Works well for FAQs, forms, guides |
| TXT | Tertiary | Plain text extracts from web pages |
| HTML | Tertiary | Saved web pages (e.g., department pages) |

---

## 3. Document Preparation Checklist

Before adding a document to the corpus:

- [ ] **Text is extractable**: Open the PDF, try to select and copy text. If it's a scanned image, run OCR first (using Tesseract or Adobe Acrobat)
- [ ] **Language is consistent**: Documents should be in Romanian, English, or both. Mixed-language docs are fine
- [ ] **File is named descriptively**: Use `topic_subtopic_year.pdf` format (e.g., `calendarul_academic_2025_2026.pdf`), not `scan001.pdf`
- [ ] **Content is current**: Outdated documents will produce wrong answers. Only include the latest version
- [ ] **No personal data**: Remove any student lists, grade sheets, or documents containing student names/IDs
- [ ] **File size is reasonable**: Very large PDFs (>50MB) should be split into chapters
- [ ] **Formatting is clean**: Remove headers/footers that repeat on every page if possible (they create noise in chunking)

---

## 4. OCR for Scanned Documents

If a PDF contains scanned images rather than selectable text:

```bash
# Using Tesseract OCR (install: apt install tesseract-ocr tesseract-ocr-ron)
# Convert PDF pages to images, then OCR
pdftoppm -jpeg -r 300 scanned_document.pdf page
for img in page-*.jpg; do
  tesseract "$img" "${img%.jpg}" -l ron+eng
done
# Combine text files into one
cat page-*.txt > extracted_document.txt
```

Alternatively, use Adobe Acrobat's built-in OCR or an online tool before adding to the corpus.

---

## 5. Web Page Capture

For information that lives on the ULBS website:

```bash
# Simple text extraction from a web page
curl -s "https://www.ulbsibiu.ro/page" | \
  python3 -c "
import sys
from html.parser import HTMLParser
class P(HTMLParser):
    def __init__(self):
        super().__init__()
        self.text = []
        self.skip = False
    def handle_starttag(self, tag, attrs):
        if tag in ('script','style','nav','footer','header'):
            self.skip = True
    def handle_endtag(self, tag):
        if tag in ('script','style','nav','footer','header'):
            self.skip = False
    def handle_data(self, data):
        if not self.skip:
            self.text.append(data.strip())
p = P()
p.feed(sys.stdin.read())
print('\n'.join(t for t in p.text if t))
" > ulbs_page_name.txt
```

---

## 6. Quality Validation

After preparing the corpus, verify it works with the ingestion pipeline:

1. Run the ingestion workflow in n8n
2. Check Qdrant for chunk count: `curl http://localhost:6333/collections/ulbs_documents`
3. Test with 5 known questions that should match specific documents
4. Verify source citations point to the correct documents
5. Check for chunking issues (answers that seem cut off may indicate bad chunk boundaries)

---

## 7. Maintenance Schedule

| Task | Frequency | Owner |
|------|-----------|-------|
| Add new academic calendar | Annually (September) | Member 5 |
| Update tuition fees | Annually (July) | Member 5 |
| Re-check forms and templates | Bi-annually | Member 5 |
| Remove outdated documents | Before each re-ingestion | Member 5 |
| Re-run ingestion pipeline | After any document update | Member 2 |
| Verify search quality | After re-ingestion | Member 2 + Member 5 |

---

*Member 5 is responsible for keeping this corpus current and clean. The chatbot can only be as good as its source documents.*

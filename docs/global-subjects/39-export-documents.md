# 39. Export et génération de documents (PDF, CSV, Excel)

> 🎯 **Résumé.** Un petit CSV peut être streamé directement dans la réponse, mais un export de plusieurs centaines de milliers de lignes doit passer par un job, un fichier en stockage objet et une URL pré-signée à durée limitée. Les gabarits HTML/CSS simplifient la maintenance des PDF, tandis que les cellules commençant par `=`, `+`, `-` ou `@` doivent être neutralisées. Le piège : charger tout le résultat en mémoire et exposer un fichier sans expiration ni contrôle d'accès.

---

## 🧭 1. Contexte

**Principe** : petits exports = synchrone ; gros exports ou génération lourde = **asynchrone** (job + notification/lien de téléchargement, pattern « fire and poll » du sujet 22).

| Format | Spring | Django | Laravel | Node |
|---|---|---|---|---|
| CSV | OpenCSV / écriture en streaming | module `csv` + `StreamingHttpResponse` | `fputcsv` / `streamDownload` | `csv-stringify` |
| Excel | Apache POI | `openpyxl`, `XlsxWriter` | Laravel Excel (Maatwebsite) | `exceljs` |
| PDF | OpenPDF, JasperReports, Thymeleaf + moteur HTML→PDF | WeasyPrint, ReportLab | DomPDF, Snappy | Puppeteer, pdfkit |

**Bonnes pratiques**
- **Streamer** les gros CSV (ne pas charger tout en mémoire) et itérer par lots côté base.
- Stocker le fichier généré dans un stockage objet (S3) et renvoyer une **URL pré-signée** à durée limitée.
- Gabarits HTML + CSS d'impression pour les PDF (plus faciles à maintenir que du dessin manuel).
- Échapper les cellules commençant par `=`, `+`, `-`, `@` dans les exports CSV/Excel (injection de formules).

---

## 💻 2. Exemples techniques

Ce CSV Django est produit en flux avec `iterator(chunk_size=1000)` ; `safe_cell` neutralise une tentative d'injection de formule en préfixant la valeur par une apostrophe :

```python
import csv

from django.http import StreamingHttpResponse

from .models import Invoice


class Echo:
    def write(self, value):
        return value


def safe_cell(value):
    text = "" if value is None else str(value)
    return "'" + text if text[:1] in ("=", "+", "-", "@") else text


def export_rows():
    yield ["id", "email", "montant_eur"]
    for invoice in Invoice.objects.order_by("id").iterator(chunk_size=1000):
        yield [invoice.id, safe_cell(invoice.email), f"{invoice.total_cents / 100:.2f}"]


def export_csv(request):
    writer = csv.writer(Echo())
    response = StreamingHttpResponse(
        (writer.writerow(row) for row in export_rows()),
        content_type="text/csv; charset=utf-8",
    )
    response["Content-Disposition"] = 'attachment; filename="invoices.csv"'
    return response
```

Pour un gros export, le job Celery écrit le fichier dans S3 et l'enregistre comme prêt ; le téléchargement ne renvoie qu'une URL pré-signée valable 15 minutes et vérifiée contre le tenant :

```python
import csv
import os
import tempfile

import boto3
from celery import shared_task
from django.conf import settings
from django.http import JsonResponse

from .models import Export, Invoice


@shared_task
def build_invoice_export(export_id: int) -> None:
    export = Export.objects.get(pk=export_id)
    with tempfile.NamedTemporaryFile(mode="w", suffix=".csv", delete=False) as handle:
        writer = csv.writer(handle)
        writer.writerow(["id", "total_cents"])
        for invoice in Invoice.objects.filter(period=export.period).iterator(chunk_size=1000):
            writer.writerow([invoice.id, invoice.total_cents])
        path = handle.name

    key = f"exports/{export.tenant_id}/{export.id}.csv"
    boto3.client("s3", endpoint_url=settings.S3_ENDPOINT).upload_file(
        path, settings.S3_BUCKET, key
    )
    os.unlink(path)

    export.status = "ready"
    export.object_key = key
    export.save(update_fields=["status", "object_key"])


def download_export(request, export_id: int):
    export = Export.objects.get(pk=export_id, tenant_id=request.user.tenant_id)
    url = boto3.client("s3", endpoint_url=settings.S3_ENDPOINT).generate_presigned_url(
        "get_object",
        Params={"Bucket": settings.S3_BUCKET, "Key": export.object_key},
        ExpiresIn=900,
    )
    return JsonResponse({"url": url})
```

Un PDF WeasyPrint est rendu depuis un gabarit HTML, avec une feuille de style d'impression qui répète l'en-tête des tableaux et évite les coupures de lignes :

```python
from django.conf import settings
from django.template.loader import render_to_string
from weasyprint import HTML


def render_invoice_pdf(invoice) -> bytes:
    html = render_to_string("invoices/invoice.html", {"invoice": invoice})
    return HTML(string=html, base_url=settings.STATIC_ROOT).write_pdf()
```

La feuille de style d'impression assure la pagination et évite les lignes coupées :

```css
@page {
  size: A4;
  margin: 18mm;
}

body {
  font-family: "DejaVu Sans", sans-serif;
  font-size: 10pt;
}

table {
  width: 100%;
  border-collapse: collapse;
}

thead {
  display: table-header-group;
}

tr {
  break-inside: avoid;
}
```

---

## 🔗 3. Voir aussi

- [Sujet 22 — Jobs asynchrones et files](22-jobs-asynchrones-files.md)
- [Sujet 24 — Logging et observabilité](24-logging-observabilite.md)
- [Sujet 26 — Upload de fichiers](26-upload-fichiers.md)
- [Stockage objet, partie 1](../backend-from-first-principles/23-object-storage-part1.md)

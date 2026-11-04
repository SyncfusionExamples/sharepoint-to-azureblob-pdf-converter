# SharePoint to Azure Blob PDF Converter

An Azure Functions solution (.NET 10, isolated worker model) that retrieves Office documents (`.docx`, `.pptx`) from a Microsoft SharePoint document library, converts them into PDF format using Syncfusion DocIO and Presentation renderers, and uploads the resulting PDFs into Azure Blob Storage.

---

## Architecture & Workflow Overview

```
┌─────────────────────────┐
│   HTTP Trigger Request  │  (GET/POST to /api/ConvertDocuments)
└────────────┬────────────┘
             │
             ▼
┌─────────────────────────┐
│   SharePoint Service    │  Authenticates with Azure AD (Microsoft Graph API)
│  (Microsoft Graph API)  │  Fetches .docx and .pptx files from target document library/folder
└────────────┬────────────┘
             │ Stream
             ▼
┌─────────────────────────┐
│      PDF Converter      │  Converts .docx using Syncfusion DocIO Renderer
│  (Syncfusion Libraries) │  Converts .pptx using Syncfusion Presentation Renderer
└────────────┬────────────┘
             │ PDF Stream
             ▼
┌─────────────────────────┐
│   Azure Blob Storage    │  Uploads generated PDFs to the specified container
│         Service         │  Sets Content-Type: application/pdf
└─────────────────────────┘
```

### Detailed Workflow Steps

1. **Triggering**: An HTTP request (GET or POST) hits the `ConvertDocuments` endpoint in the Azure Function.
2. **License Registration**: Validates and registers the `SYNCFUSION_LICENSE_KEY` from environment variables if configured.
3. **SharePoint Document Retrieval**:
   - Uses `ClientSecretCredential` with Azure AD tenant ID, client ID, and client secret to authenticate with Microsoft Graph API.
   - Resolves the SharePoint site, drive (document library), and target folder path.
   - Iterates through the drive items, filtering for supported documents (`.docx`, `.pptx`) and ignoring temporary files (starting with `~$`).
   - Downloads document streams asynchronously.
4. **Document Conversion**:
   - Converts Word documents (`.docx`) to PDF via `Syncfusion.DocIORenderer`.
   - Converts PowerPoint presentations (`.pptx`) to PDF via `Syncfusion.PresentationRenderer`.
   - Renames file extensions to `.pdf`.
5. **Azure Blob Storage Upload**:
   - Ensures the target blob container exists (`CreateIfNotExistsAsync`).
   - Uploads each PDF file with HTTP headers set to `ContentType = "application/pdf"`.
6. **Execution Result**: Returns an HTTP JSON response summarizing the total succeeded and failed conversions:
   ```json
   {
     "converted": 5,
     "failed": 0
   }
   ```

---

## Project Structure

```
sharepoint-to-azureblob-pdf-converter/
└── SharePoint-To-Blob-Pdf-Converter/
    └── .NET/
        ├── SharePoint-To-Blob-Pdf-Converter.slnx
        └── SharePoint-To-Blob-Pdf-Converter/
            ├── Configuration/
            │   └── AppSettings.cs          # Azure AD, SharePoint, and Blob Storage configuration models
            ├── Services/
            │   ├── SharePointService.cs    # Microsoft Graph integration for fetching documents
            │   ├── PdfConverter.cs         # Syncfusion DocIO and Presentation to PDF conversion
            │   └── BlobStorageService.cs   # Azure Blob container & upload management
            ├── Function1.cs                # HTTP-triggered Azure Function (ConvertDocuments)
            ├── Program.cs                  # Function app entry point & OpenTelemetry setup
            ├── host.json                   # Azure Functions host settings
            ├── SharePoint-To-Blob-Pdf-Converter.csproj
            └── Properties/
                └── launchSettings.json
```

---

## Prerequisites

- [.NET 10 SDK](https://dotnet.microsoft.com/)
- [Azure Functions Core Tools v4](https://learn.microsoft.com/azure/azure-functions/functions-run-local)
- Azure Subscription with:
  - Azure App Registration (with Microsoft Graph permissions, e.g. `Files.Read.All` or `Sites.Read.All` application permissions with admin consent)
  - Azure Blob Storage Account and target Container
  - Access to target SharePoint site and document library
- [Syncfusion License Key](https://help.syncfusion.com/common/essential-studio/licensing/how-to-generate) (optional for evaluation, recommended for production)

---

## Configuration

Set the configuration values in `Configuration/AppSettings.cs` or adapt them to environment / `local.settings.json`:

### 1. Azure AD Settings
- **TenantId**: Azure Active Directory Tenant ID.
- **ClientId**: Azure AD Application (Client) ID.
- **ClientSecret**: Client secret created for the app registration.

### 2. SharePoint Settings
- **SiteUrl**: Absolute URL of your SharePoint site (e.g. `https://yourtenant.sharepoint.com/sites/YourSite`).
- **LibraryName**: Name of the document library (e.g. `Documents` or `Shared Documents`).
- **FolderPath**: Path to the subfolder containing documents (e.g. `InputFiles` or leave blank/root).

### 3. Azure Blob Storage Settings
- **ConnectionString**: Azure Storage connection string.
- **ContainerName**: Target blob container name where converted PDFs will be saved.


---

## Running the Application Locally

1. Navigate to the project folder:
   ```bash
   cd SharePoint-To-Blob-Pdf-Converter/.NET/SharePoint-To-Blob-Pdf-Converter
   ```

2. Restore dependencies and build the project:
   ```bash
   dotnet build
   ```

3. Start the Azure Functions runtime:
   ```bash
   func start
   ```

4. Trigger the function:
   Send an HTTP GET or POST request:
   ```bash
   curl -X POST http://localhost:7071/api/ConvertDocuments
   ```

---

## Supported File Formats

| Source Extension | Target Extension | Engine |
| ---------------- | ---------------- | ------ |
| `.docx`          | `.pdf`           | Syncfusion DocIO & DocIORenderer |
| `.pptx`          | `.pdf`           | Syncfusion Presentation & PresentationRenderer |
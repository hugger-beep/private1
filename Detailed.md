## Detailed System Architecture

```mermaid
graph TD
    subgraph "Client Layer"
        DIWebApp[DI WebApp]
    end
    
    subgraph "API Layer"
        APIGateway[API Gateway]
        APIAuth[API Authorizer Lambda]
    end
    
    subgraph "Authentication Layer"
        AuthMicro[Authentication Microservice Lambda]
        XACIAuth[XACI Authentication API]
    end
    
    subgraph "Reporting Layer"
        ReportMicro[Report Microservice]
        QS[QuickSight]
    end
    
    subgraph "Data Layer"
        RDSProxy[RDS Proxy]
        RDS[RDS Database]
        DW[Glue Data Warehouse]
        
        subgraph "ETL Processes"
            TransETL[Transaction Data Glue ETL]
            MasterETL[Master Data Glue ETL]
            SecureSync[Secure Access Data Sync]
            ManualUpdate[Manual Exam Data Update]
        end
    end
    
    subgraph "External Systems"
        XACTDataAPI[XACT Data APIs]
    end
    
    % Client to API connections
    DIWebApp --> APIGateway
    
    % API Gateway connections
    APIGateway --> AuthMicro
    APIGateway --> ReportMicro
    APIGateway --> APIAuth
    
    % Authentication connections
    AuthMicro --> XACIAuth
    AuthMicro --> RDSProxy
    APIAuth --> RDSProxy
    
    % Reporting connections
    ReportMicro --> QS
    ReportMicro --> APIAuth
    
    % Data layer connections
    RDSProxy --> RDS
    RDS <--> SecureSync
    SecureSync --> DW
    TransETL --> DW
    MasterETL --> RDS
    ManualUpdate --> DW
```
    % External connections
    XACTDataAPI --> TransETL

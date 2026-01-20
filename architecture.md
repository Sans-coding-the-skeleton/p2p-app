# ESSENTIALS BANK NODE - Architecture

## Class Diagram

```mermaid
classDiagram
    %% Relations
    Main ..> AppConfig : uses
    Main ..> Logger : uses
    Main --> TcpServer : starts
    
    TcpServer ..> ConnectionHandler : spawns
    ConnectionHandler ..> CommandProcessor : uses
    
    CommandProcessor --> CommandParser : uses
    CommandProcessor --> CommandRouter : uses
    
    CommandRouter --> BankService : routes explicit local commands
    CommandRouter --> ProxyService : routes remote commands (Essentials)
    
    BankService --> AccountRepository : persists data
    
    ProxyService ..> TcpClient : uses
    
    AccountRepository --> StorageEngine : reads/writes
    
    %% NETWORK LAYER
    namespace Network_Layer {
        class TcpServer {
            -int port
            -bool running
            +start()
            +stop()
            +accept_connections()
        }
        
        class ConnectionHandler {
            -Socket socket
            -int timeout
            +handle()
            -send_response(String)
        }

        class TcpClient {
            +send_command(String ip, String command) String
            -connect(String ip)
        }
    }

    %% PROTOCOL LAYER
    namespace Protocol_Layer {
        class CommandParser {
            +parse(String raw_data) Command
            -validate_format()
        }
        
        class CommandProcessor {
            +process(String input) String
            -format_error(String msg)
        }
        
        class CommandRouter {
            -String local_ip
            +route(Command cmd) Result
            -is_local(String target_ip) bool
        }
    }

    %% BUSINESS LOGIC LAYER
    namespace Business_Logic_Layer {
        class BankService {
            -AccountRepository repo
            +get_bank_code_BC()
            +create_account_AC()
            +deposit_AD(int account, int amount)
            +withdraw_AW(int account, int amount)
            +get_balance_AB(int account)
            +remove_account_AR(int account)
            +get_total_amount_BA()
            +get_client_count_BN()
        }
        
        class ProxyService {
            -int timeout
            +forward_command(String target_ip, String command) String
        }
    }

    %% DATA LAYER
    namespace Data_Layer {
        class AccountRepository {
            -String data_file_path
            -Map~int, Account~ cache
            +find_by_id(int id) Account
            +save(Account acc)
            +delete(int id)
            +find_all() List~Account~
            +load_from_disk()
            +persist_to_disk()
        }
        
        class Account {
            -int account_number
            -long balance
            +deposit(long amount)
            +withdraw(long amount)
            +get_balance() long
        }
        
        class StorageEngine {
            <<Interface>>
            +read()
            +write()
        }
    }

    %% CROSS-CUTTING CONCERNS
    namespace Cross_Cutting {
        class AppConfig {
            +get_port()
            +get_local_ip()
            +get_timeout()
        }
        
        class Logger {
            +log_info(String)
            +log_error(String)
            +log_traffic(String source, String dest, String msg)
        }
    }
```

## Description of Layers

1.  **Network Layer**: Handles TCP connectivity. `TcpServer` accepts connections, `ConnectionHandler` processes them, and `TcpClient` is used by the Proxy optimization to forward commands.
2.  **Protocol Layer**: Responsible for parsing text commands (`CommandParser`) and deciding where they go (`CommandRouter`). The router distinguishes between local commands (for `BankService`) and remote commands (for `ProxyService`).
3.  **Business Logic Layer**:
    *   `BankService`: Implements the bank logic (accounts, deposits, etc.) for *this* node.
    *   `ProxyService`: Forwards commands to other nodes if the generic bank code (IP) doesn't match the local one.
4.  **Data Layer**: Handles persistence. `AccountRepository` saves/loads accounts to disk so data is not lost on restart.

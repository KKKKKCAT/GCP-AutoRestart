# Google GCP 自動開機 Python 腳本

文章同步在：https://kkcat.blog/posts/google-gcp-spot-script-2024/

![](https://kkcat.blog/_astro/Google-GCP-spot-script-2024-01-poster01.Bcr0t35w_1NSmSf.webp)


**介紹**

本文將介紹一個Python腳本，自動檢查並啟動處於非運行狀態的GCE(Google Cloud Compute Engine)實例，以確保其持續可用。

**需求**

- 在開始之前，請確保您已經具備以下條件：

- 安裝了Google Cloud SDK。

- 配置了正確的GCP項目，並下載了服務賬號的JSON密鑰文件。

- 在您的系統中安裝了google-cloud-compute庫：
```
pip install google-cloud-compute
```

**腳本功能**

這個Python腳本將執行以下任務：

列出所有區域中的所有GCE實例。
檢查每個實例的狀態。
啟動所有處於非運行狀態的實例。
自行每3-5分鐘重覆執行上述任務，以確保實例的持續可用性。

***設置服務賬號密鑰：***

1. API和服務 -> 憑證 -> 服務帳戶(默認那個即可) -> 金鑰 -> 新增金鑰(json檔)
2. project ID看圖取

![](https://kkcat.blog/_astro/gcp_project_id_01.Ciyh8-mT_Z1zX06J.webp)

**腳本代碼**
```
import os
import time
from google.cloud import compute_v1

def start_instance_if_not_running(project_id, zone, instance_name):
    try:
        instance_client = compute_v1.InstancesClient()
        instance = instance_client.get(project=project_id, zone=zone, instance=instance_name)

        if instance.status != "RUNNING":
            print(f"Instance {instance_name} is not running. Starting the instance.")
            operation = instance_client.start(project=project_id, zone=zone, instance=instance_name)
            wait_for_operation(project_id, zone, instance_name, operation.name)
            instance = instance_client.get(project=project_id, zone=zone, instance=instance_name)
            print(f"Instance {instance_name} is now {instance.status}.")
        else:
            print(f"Instance {instance_name} is already running.")

        return instance.status
    except Exception as e:
        print(f"Error in start_instance_if_not_running for instance {instance_name} in project {project_id}: {e}")
        return "ERROR"

def wait_for_operation(project_id, zone, instance_name, initial_operation_name, timeout=300):
    current_operation_name = initial_operation_name  # 初始化变量
    try:
        operation_client = compute_v1.ZoneOperationsClient()
        instance_client = compute_v1.InstancesClient()
        start_time = time.time()

        while True:
            result = operation_client.get(project=project_id, zone=zone, operation=current_operation_name)
            current_status = result.status
            print(f"Waiting for operation {current_operation_name} to complete. Current status: {current_status}")
            if current_status == "DONE":
                if result.error:
                    raise Exception(f"Error during operation: {result.error}")
                print("Operation finished successfully.")
                break
            if time.time() - start_time > timeout:
                raise TimeoutError(f"Operation {current_operation_name} timed out.")
            instance = instance_client.get(project=project_id, zone=zone, instance=instance_name)
            if instance.status == "RUNNING":
                print(f"Instance {instance_name} has started running.")
                break
            if instance.status == "PROVISIONING":
                current_operation_name = instance.last_start_operation
                print(f"Operation name has changed to {current_operation_name}")
            time.sleep(5)
    except Exception as e:
        print(f"Error in wait_for_operation for operation {current_operation_name} in project {project_id}: {e}")
        raise

def list_and_start_instances(project_id):
    try:
        compute_client = compute_v1.InstancesClient()
        zones_client = compute_v1.ZonesClient()

        result_table = f"{'Instance name'.ljust(20)} | {'Zone'.ljust(20)} | Status\n\n"

        zones = zones_client.list(project=project_id)
        instances_count = 0

        for zone in zones:
            zone_name = zone.name
            request = compute_v1.ListInstancesRequest(project=project_id, zone=zone_name)
            response = compute_client.list(request=request)

            for instance in response:
                instances_count += 1
                status = start_instance_if_not_running(project_id, zone_name, instance.name)
                result_table += f"{instance.name} | {zone_name} | {status}\n"

        print("\n" + '                                                      ' + '\n')
        print(result_table)
        print(f"Total number of instances: {instances_count}")
        print('\n' + '                                                      ')
    except Exception as e:
        print(f"Error in list_and_start_instances for project {project_id}: {e}")

if __name__ == "__main__":
    credentials_and_projects = [
        {"credentials": "GCP.json", "project_id": "xxxxxxxxx"},
    ]

    for item in credentials_and_projects:
        try:
            credentials_file = item["credentials"]
            if not os.path.isfile(credentials_file):
                print(f"Credentials file {credentials_file} does not exist.")
                continue
            os.environ["GOOGLE_APPLICATION_CREDENTIALS"] = credentials_file
            project_id = item["project_id"]
            list_and_start_instances(project_id)
        except Exception as e:
            print(f"Error processing project {item['project_id']}: {e}")

    # 清理环境变量
    os.environ["GOOGLE_APPLICATION_CREDENTIALS"] = ""

```

**設置**
1. 設置服務賬號密鑰json和project_id：
  ```
  {"credentials": "GCP.json", "project_id": "xxxxxxxxxxxxx"},
  ```
2. 使用命令 crontab -e 打開 crontab 編輯器添加如下行來每5分鐘執行一次您的腳本：
```
*/5 * * * * python3 /path/to/your/gcp.py >> /path/to/logfile.log 2>&1
```


**結論**

通過這個Python腳本，可以自動化管理GCP中的虛擬機實例，確保它們始終處於運行狀態。
如果您有更多的實例或區域需要管理，可以根據需要擴展這個腳本。
   


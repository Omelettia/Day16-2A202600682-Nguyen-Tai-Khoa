# Báo cáo Lab 16
## Môi trường thực hiện

Môi trường local được sử dụng:

```text
Windows + WSL Ubuntu
AWS CLI v2
Terraform
SSH key pair: lab-key / lab-key.pub
Region AWS: us-east-1
```
## Vấn đề phát sinh khi chuẩn bị SSH key

Terraform yêu cầu file public key sau tồn tại trong thư mục Terraform:

```text
lab-key.pub
```

Tuy nhiên trong quá trình làm lab, file này chưa có sẵn nên Terraform không thể tạo AWS key pair ở lần chạy đầu.

Cách xử lý là tạo SSH key pair thủ công trong thư mục Terraform:

```bash
ssh-keygen -t rsa -b 4096 -f lab-key -N ""
chmod 400 lab-key
```

Sau khi tạo `lab-key` và `lab-key.pub`, Terraform có thể tiếp tục tạo hạ tầng.

---

## Vấn đề AWS quota cho GPU

Ban đầu tài khoản AWS chưa có quota để chạy GPU instance loại G/VT trong region `us-east-1`.

Quota cần thiết cho cấu hình ban đầu là:

```text
Running On-Demand G and VT instances: 4 vCPUs
```

Cấu hình `g4dn.xlarge` cần 4 vCPU, nên phải request quota đúng region:

```text
Region: us-east-1 / US East (N. Virginia)
Quota: Running On-Demand G and VT instances
Requested value: 4
```

Một điểm cần chú ý là quota được duyệt theo từng region. Nếu quota được duyệt ở region khác, ví dụ Sydney, thì không áp dụng được cho Terraform đang chạy ở `us-east-1`.

---

##Lần triển khai GPU ban đầu với `g4dn.xlarge`

Cấu hình ban đầu của bài lab sử dụng:

```hcl
instance_type = "g4dn.xlarge"
```

Instance này dùng GPU NVIDIA Tesla T4.

Sau khi triển khai, hệ thống có thể tạo được hạ tầng AWS gồm VPC, Bastion Host, GPU node, NAT Gateway và ALB. Tuy nhiên, inference bằng model `google/gemma-4-E2B-it` không chạy thành công trên cấu hình này.

Trong quá trình kiểm tra, container vLLM có thể khởi động và endpoint health check có lúc trả về:

```text
200 OK
```

Nhưng khi gửi request inference thật, vLLM bị lỗi runtime. Log của container cho thấy lỗi chính:

```text
triton.runtime.errors.OutOfResources:
out of resource: shared memory,
Required: 98304,
Hardware limit: 65536
```

Điều này cho thấy vấn đề không nằm ở networking hay ALB nữa. Model đã được load, nhưng khi inference, kernel Triton mà vLLM sử dụng yêu cầu dung lượng shared memory lớn hơn giới hạn phần cứng của GPU Tesla T4.

Kết luận cho lần thử ban đầu:

```text
Cấu hình g4dn.xlarge / Tesla T4 không chạy inference thành công với model google/gemma-4-E2B-it và Docker image vLLM hiện tại.
```

---

## Vấn đề Docker image không tự tải thành công khi instance mới boot

Script `user_data.sh` được dùng để tự động cài đặt và chạy Docker container vLLM khi GPU instance khởi động.

Tuy nhiên trong thực tế, ở thời điểm instance vừa boot, lệnh pull Docker image bị timeout khi kết nối Docker Hub:

```text
failed to resolve reference "docker.io/vllm/vllm-openai:latest"
dial tcp ...:443: i/o timeout
```

Vì vậy container `vllm` không được tạo tự động sau khi Terraform apply xong.

Để xử lý, tôi phải SSH vào GPU node thông qua Bastion Host và chạy lại thủ công các bước Docker.

Lệnh pull image thủ công:

```bash
sudo docker pull vllm/vllm-openai:latest
```

Sau đó chạy container thủ công:

```bash
HF_TOKEN=$(sudo sed -n 's/^export HF_TOKEN="\([^"]*\)"/\1/p' /var/lib/cloud/instance/user-data.txt)

sudo docker rm -f vllm 2>/dev/null || true

sudo docker run -d --name vllm \
  --runtime nvidia --gpus all \
  --restart unless-stopped \
  -e HF_TOKEN="$HF_TOKEN" \
  -v /opt/huggingface:/root/.cache/huggingface \
  -p 8000:8000 \
  --ipc=host \
  vllm/vllm-openai:latest \
  --model google/gemma-4-E2B-it \
  --max-model-len 2048 \
  --gpu-memory-utilization 0.90 \
  --host 0.0.0.0
```

Sau đó theo dõi log:

```bash
sudo docker logs -f vllm
```

Khi thấy server khởi động xong, kiểm tra local health check:

```bash
curl -v http://localhost:8000/health
```

---

## Nâng cấp GPU sang `g5.xlarge`

Do `g4dn.xlarge` dùng Tesla T4 không chạy inference thành công, tôi thay đổi instance type trong Terraform từ:

```hcl
instance_type = "g4dn.xlarge"
```

thành:

```hcl
instance_type = "g5.xlarge"
```

Lý do nâng cấp:

```text
g5.xlarge sử dụng GPU NVIDIA A10G mới hơn và có tài nguyên GPU phù hợp hơn so với Tesla T4.
```

Sau khi đổi sang `g5.xlarge`, hạ tầng được apply lại bằng Terraform. GPU instance mới chạy thành công, Docker image được pull thủ công, container vLLM được khởi động thủ công, và model load thành công.

---

## Kiểm thử API thành công

Sau khi vLLM chạy ổn định trên `g5.xlarge`, tôi gọi API thông qua Application Load Balancer.

Lệnh kiểm thử thành công:

```bash
ALB=$(terraform output -raw alb_dns_name)

curl -X POST "http://$ALB/v1/chat/completions" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "google/gemma-4-E2B-it",
    "messages": [
      {"role": "system", "content": "Bạn là một trợ lý AI hữu ích."},
      {"role": "user", "content": "Hãy giải thích Bastion Host trong AWS là gì?"}
    ],
    "max_tokens": 150
  }'
```

Kết quả trả về thành công có nội dung tiếng Việt, bắt đầu bằng:

```text
Chào bạn, tôi rất sẵn lòng giải thích về Bastion Host trong AWS.
```

Điều này xác nhận luồng triển khai hoạt động đầy đủ:

```text
Client → Application Load Balancer → GPU node trong private subnet → Docker vLLM → model inference
```

---

## Thời gian cold start

Thời gian cold start của lần triển khai thành công là khoảng:

```text
21 phút
```

Thời gian này bao gồm:

- Terraform tạo hạ tầng AWS
- EC2 GPU instance khởi động
- SSH vào instance để kiểm tra
- Docker image được pull
- Model được tải / load
- vLLM server khởi động
- Gửi request API đầu tiên thành công

Lần thử với `g4dn.xlarge` mất nhiều thời gian hơn do phải debug lỗi runtime và xác định nguyên nhân liên quan đến giới hạn shared memory của Tesla T4.

---

## Ghi chú về chi phí và Billing

Các tài nguyên có thể phát sinh chi phí trong bài lab:

```text
g5.xlarge GPU instance
Bastion EC2 instance
NAT Gateway
Application Load Balancer
Elastic IP
EBS volume
```

Sau khi hoàn thành screenshot API thành công, cần destroy hạ tầng ngay để tránh phát sinh thêm chi phí:

```bash
terraform destroy
```

AWS Billing và Cost Explorer không cập nhật real-time. Do đó, sau vài giờ có thể vẫn chưa thấy chi tiết chi phí trong trang Billing. Trong trường hợp đó, có thể chụp màn hình Billing/Credits/Budget hiện có và ghi chú rằng dữ liệu billing của AWS có độ trễ cập nhật.

---

## Cleanup sau khi hoàn thành

Sau khi lấy screenshot kết quả API thành công, tôi chạy:

```bash
terraform destroy
```

và chờ đến khi Terraform báo:

```text
Destroy complete!
```

Sau đó kiểm tra lại các tài nguyên còn chạy:

```bash
aws ec2 describe-instances \
  --region us-east-1 \
  --filters "Name=instance-state-name,Values=pending,running,stopping,stopped" \
  --query "Reservations[].Instances[].{ID:InstanceId,Type:InstanceType,State:State.Name}" \
  --output table

aws ec2 describe-nat-gateways \
  --region us-east-1 \
  --filter "Name=state,Values=available,pending" \
  --query "NatGateways[].{ID:NatGatewayId,State:State,Vpc:VpcId}" \
  --output table
```

Mục tiêu là đảm bảo không còn GPU instance, NAT Gateway hoặc tài nguyên đắt tiền nào còn chạy.

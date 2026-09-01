kubectl create secret docker-registry ecr-secret \
  --docker-server=925963414980.dkr.ecr.ap-south-1.amazonaws.com \
  --docker-username=AWS \
  --docker-password="$(aws ecr get-login-password --region ap-south-1)" \
  -n feedback

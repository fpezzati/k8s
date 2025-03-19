10. Configure Horizontal Pod Autoscaler (HPA)
Instructions:

    Create a Horizontal Pod Autoscaler (HPA) for the deployment named api-deployment located in the api namespace.
    The HPA should scale the deployment based on a custom metric named requests_per_second, targeting an average value of 1000 requests per second across all pods.
    Set the minimum number of replicas to 1 and the maximum to 20.

Note: Deployment named api-deployment is available in api namespace.

LEFT UNTOUCHED
kubectl autoscale deployment api-deployment --min=1 --max=20 --dry-run=client -o yaml > hpa-api-deployment.yaml

Create a custom metric:


Modify hpa-api-deployment.yaml to add metrics

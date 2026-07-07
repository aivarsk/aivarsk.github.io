You don't need them

- Use celery settings to run tasks immediately
- for external services, mock on the network (dummy http server): this might be more work, but easier to setup with real responses from external service. And you are testing 100% of your service without skipping some strange serialization, header, domain lookup parts

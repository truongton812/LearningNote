```
resource "aws_api_gateway_rest_api" "ticketing-api" {
  name = "ticketing-api"
  description = "Ticket booking entrypoint"

  endpoint_configuration {
    types = ["Edge-optimized"]
  }
}

resource "aws_api_gateway_resource" "root" {
  rest_api_id = aws_api_gateway_rest_api.ticketing-api.id
  parent_id = aws_api_gateway_rest_api.ticketing-api.root_resource_id
  path_part = "/mypath"
}

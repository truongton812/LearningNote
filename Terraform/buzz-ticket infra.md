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


locals {
  public_subnet_map = {
    for idx, cidr in var.public_subnet_cidr_block : 
    "public-${var.az[idx]}" => {
      cidr_block      = cidr
      availability_zone = var.az[idx]
    }
  }
}

locals {
  private_subnet_map = {
    for idx, cidr in var.private_subnet_cidr_block : 
    "private-${var.az[idx]}" => {
      cidr_block      = cidr
      availability_zone = var.az[idx]
    }
  }
}


resource "aws_subnet" "public-subnet" {
  for_each          = local.public_subnet_map
  vpc_id            = aws_vpc.main.id
  cidr_block        = each.value.cidr_block
  availability_zone = each.value.availability_zone

  tags = {
    Name = each.key
  }
}

resource "aws_subnet" "private-subnet" {
  for_each          = local.private_subnet_map
  vpc_id            = aws_vpc.main.id
  cidr_block        = each.value.cidr_block
  availability_zone = each.value.availability_zone

  tags = {
    Name = each.key
  }
}

```
# ============================================================================
# VPC - Network Infrastructure
# ============================================================================

# -----------------------------------------------------------------------------
# VPC
# -----------------------------------------------------------------------------
resource "aws_vpc" "main" {
  cidr_block           = var.vpc_cidr_block
  enable_dns_support   = true
  enable_dns_hostnames = true
  tags = {
    Name = "${var.project_name}-${var.environment}-vpc"
  }
}

# -----------------------------------------------------------------------------
# Internet Gateway
# -----------------------------------------------------------------------------
resource "aws_internet_gateway" "this" {
  vpc_id = aws_vpc.main.id
  tags = {
    Name = "${var.project_name}-${var.environment}-igw"
  }
}

# -----------------------------------------------------------------------------
# Public Subnet
# -----------------------------------------------------------------------------
resource "aws_subnet" "public_subnet" {
  count             = length(var.public_subnet_cidr)
  vpc_id            = aws_vpc.main.id
  cidr_block        = var.public_subnet_cidr[count.index]
  availability_zone = var.availability_zones[count.index % length(var.availability_zones)]
  tags = {
    Name = "${var.project_name}-${var.environment}-public-subnet-${count.index + 1}"
  }
}

# Public Route table
resource "aws_route_table" "public_route_table" {
  vpc_id = aws_vpc.main.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.this.id
  }
  tags = {
    Name = "${var.project_name}-${var.environment}-public-route-table"
  }
}

# Associate public route table to public subnet
resource "aws_route_table_association" "public_associate" {
  count          = length(aws_subnet.public_subnet)
  subnet_id      = aws_subnet.public_subnet[count.index].id
  route_table_id = aws_route_table.public_route_table.id
}

# -----------------------------------------------------------------------------
# Private subnet
# -----------------------------------------------------------------------------
resource "aws_subnet" "private_subnet" {
  vpc_id            = aws_vpc.main.id
  count             = length(var.private_subnet_cidr)
  cidr_block        = var.private_subnet_cidr[count.index]
  availability_zone = var.availability_zones[count.index % length(var.availability_zones)]
  tags = {
    Name = "${var.project_name}-${var.environment}-private-subnet-${count.index + 1}"
  }
}

resource "aws_route_table" "share_rt" {
  vpc_id = aws_vpc.main.id
  dynamic "route" {
    for_each = var.create_nat_gateway && var.single_nat_gateway ? [1] : []
    content {
      cidr_block = "0.0.0.0/0"
      nat_gateway_id = aws_nat_gateway.nat_gw[0].id
    }
  }
  tags = {
    Name = "${var.project_name}-${var.environment}-private-route-table"
  }
}

resource "aws_route_table" "per_az" {
  count = var.create_nat_gateway && !var.single_nat_gateway ? length(var.public_subnet_cidr) : 0
  vpc_id = aws_vpc.main.id
  route {
    cidr_block = "0.0.0.0/0"
    nat_gateway_id = aws_nat_gateway.nat_gw[count.index].id
  }
  tags = {
    Name = "${var.project_name}-${var.environment}-private-route-table"
  }
}

# Associate private route table to private subnet
resource "aws_route_table_association" "share_private_associate" {
  for_each =  var.create_nat_gateway ? (var.single_nat_gateway ? toset(aws_subnet.private_subnet[*].id) : []) : toset(aws_subnet.private_subnet[*].id)
  subnet_id      = each.key
  route_table_id = aws_route_table.share_rt.id
}

resource "aws_route_table_association" "per_az_private_associate" {
  count          = var.create_nat_gateway && !var.single_nat_gateway ? length(var.public_subnet_cidr) : 0
  subnet_id      = aws_subnet.private_subnet[count.index].id
  route_table_id = aws_route_table.per_az[count.index].id
}

# -----------------------------------------------------------------------------
# NAT Gateway
# -----------------------------------------------------------------------------

# Elastic IP
resource "aws_eip" "eip" {
  count = var.create_nat_gateway ? (var.single_nat_gateway ? 1 : length(var.public_subnet_cidr)) : 0
  tags = {
    Name = "${var.project_name}-${var.environment}-nat-eip-${count.index + 1}"
  }
}

# NAT Gateway
resource "aws_nat_gateway" "nat_gw" {
  count         = var.create_nat_gateway ? (var.single_nat_gateway ? 1 : length(var.public_subnet_cidr)) : 0
  allocation_id = aws_eip.eip[count.index].id
  subnet_id     = aws_subnet.public_subnet[count.index].id
  tags = {
    Name = "${var.project_name}-${var.environment}-nat-gw-${count.index + 1}"
  }
}

```

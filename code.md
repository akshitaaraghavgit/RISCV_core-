# Repo Structure
riscv-core/
│
├── alu.v
├── register_file.v
├── pc.v
├── instruction_memory.v
├── data_memory.v
├── immediate_generator.v
├── instruction_decoder.v
├── control_unit.v
├── alu_input_mux.v
├── writeback_mux.v
└── next_pc_logic.v


# alu.v
module alu (
    input  wire [31:0] a,
    input  wire [31:0] b,
    input  wire [3:0] alu_control,
    output reg  [31:0] alu_result,
    output wire zero
);

    localparam ALU_ADD = 4'b0000;
    localparam ALU_SUB = 4'b0001;
    localparam ALU_AND = 4'b0010;
    localparam ALU_OR  = 4'b0011;
    localparam ALU_XOR = 4'b0100;

    always @(*) begin
        case (alu_control)

            ALU_ADD:
                alu_result = a + b;

            ALU_SUB:
                alu_result = a - b;

            ALU_AND:
                alu_result = a & b;

            ALU_OR:
                alu_result = a | b;

            ALU_XOR:
                alu_result = a ^ b;

            default:
                alu_result = 32'b0;

        endcase
    end

    assign zero = (alu_result == 32'b0);

endmodule

# register_file.v
module register_file (
    input wire clk,
    input wire reset,

    input wire [4:0] rs1,
    input wire [4:0] rs2,
    input wire [4:0] rd,

    input wire [31:0] write_data,
    input wire reg_write,

    output wire [31:0] rs1_data,
    output wire [31:0] rs2_data
);

    reg [31:0] regs [0:31];

    integer i;

    assign rs1_data = (rs1 == 0) ? 32'b0 : regs[rs1];
    assign rs2_data = (rs2 == 0) ? 32'b0 : regs[rs2];

    always @(posedge clk or posedge reset) begin

        if (reset) begin

            for (i = 0; i < 32; i = i + 1)
                regs[i] <= 32'b0;

        end

        else begin

            if (reg_write && rd != 0)
                regs[rd] <= write_data;

            regs[0] <= 32'b0;

        end

    end

endmodule

# pc.v
module pc (
    input wire clk,
    input wire reset,
    input wire [31:0] next_pc,
    output reg [31:0] pc_value
);

    always @(posedge clk or posedge reset) begin

        if (reset)
            pc_value <= 32'b0;

        else
            pc_value <= next_pc;

    end

endmodule

# instruction_memory.v
module instruction_memory (
    input wire [31:0] pc,
    output wire [31:0] instruction
);

    reg [31:0] imem [0:255];

    assign instruction = imem[pc[9:2]];

endmodule

# data_memory.v
module data_memory (
    input wire clk,

    input wire mem_read,
    input wire mem_write,

    input wire [31:0] address,
    input wire [31:0] write_data,

    output wire [31:0] read_data
);

    reg [31:0] dmem [0:255];

    assign read_data = dmem[address[9:2]];

    always @(posedge clk) begin

        if (mem_write)
            dmem[address[9:2]] <= write_data;

    end

endmodule
# immediate_generator.v
module immediate_generator (
    input wire [31:0] instruction,

    output wire [31:0] imm_i,
    output wire [31:0] imm_s,
    output wire [31:0] imm_b
);

    assign imm_i = {
        {20{instruction[31]}},
        instruction[31:20]
    };

    assign imm_s = {
        {20{instruction[31]}},
        instruction[31:25],
        instruction[11:7]
    };

    assign imm_b = {
        {19{instruction[31]}},
        instruction[31],
        instruction[7],
        instruction[30:25],
        instruction[11:8],
        1'b0
    };

endmodule
# instruction_decoder.v
module instruction_decoder (
    input wire [31:0] instruction,

    output wire [6:0] opcode,
    output wire [2:0] funct3,
    output wire [6:0] funct7,

    output wire [4:0] rs1,
    output wire [4:0] rs2,
    output wire [4:0] rd
);

    assign opcode = instruction[6:0];

    assign funct3 = instruction[14:12];

    assign funct7 = instruction[31:25];

    assign rs1 = instruction[19:15];

    assign rs2 = instruction[24:20];

    assign rd = instruction[11:7];

endmodule
# control_unit.
module control_unit (
    input wire [6:0] opcode,
    input wire [2:0] funct3,
    input wire [6:0] funct7,

    output reg reg_write,
    output reg mem_read,
    output reg mem_write,
    output reg alu_src,
    output reg mem_to_reg,
    output reg branch,

    output reg [3:0] alu_control
);

    localparam ALU_ADD = 4'b0000;
    localparam ALU_SUB = 4'b0001;
    localparam ALU_AND = 4'b0010;
    localparam ALU_OR  = 4'b0011;
    localparam ALU_XOR = 4'b0100;

    always @(*) begin

        reg_write   = 0;
        mem_read    = 0;
        mem_write   = 0;
        alu_src     = 0;
        mem_to_reg  = 0;
        branch      = 0;
        alu_control = ALU_ADD;

        case (opcode)

            // R-type
            7'b0110011: begin

                reg_write = 1;
                alu_src = 0;

                case (funct3)

                    3'b000: begin
                        if (funct7 == 7'b0100000)
                            alu_control = ALU_SUB;
                        else
                            alu_control = ALU_ADD;
                    end

                    3'b111:
                        alu_control = ALU_AND;

                    3'b110:
                        alu_control = ALU_OR;

                    3'b100:
                        alu_control = ALU_XOR;

                    default:
                        alu_control = ALU_ADD;

                endcase
            end

            // ADDI
            7'b0010011: begin

                reg_write = 1;
                alu_src = 1;
                alu_control = ALU_ADD;

            end

            // LW
            7'b0000011: begin

                reg_write = 1;
                mem_read = 1;
                alu_src = 1;
                mem_to_reg = 1;
                alu_control = ALU_ADD;

            end

            // SW
            7'b0100011: begin

                mem_write = 1;
                alu_src = 1;
                alu_control = ALU_ADD;

            end

            // BEQ
            7'b1100011: begin

                branch = 1;
                alu_src = 0;
                alu_control = ALU_SUB;

            end

            default: begin

            end

        endcase

    end

endmodule
# alu_input_mux.v
module alu_input_mux (
    input wire [31:0] rs2_data,
    input wire [31:0] imm_i,
    input wire [31:0] imm_s,

    input wire alu_src,
    input wire [6:0] opcode,

    output wire [31:0] alu_b
);

    assign alu_b = alu_src ?
                   ((opcode == 7'b0100011) ? imm_s : imm_i) :
                   rs2_data;

endmodule
# writeback_mux.v
module writeback_mux (
    input wire [31:0] alu_result,
    input wire [31:0] memory_data,
    input wire mem_to_reg,

    output wire [31:0] writeback_data
);

    assign writeback_data =
        mem_to_reg ? memory_data : alu_result;

endmodule
# next_pc_logic.v
module next_pc_logic (
    input wire [31:0] pc,
    input wire [31:0] imm_b,

    input wire branch,
    input wire alu_zero,

    output wire [31:0] next_pc
);

    wire [31:0] pc_plus_4;
    wire [31:0] branch_target;

    assign pc_plus_4 = pc + 32'd4;

    assign branch_target = pc + imm_b;

    assign next_pc =
        (branch && alu_zero) ?
        branch_target :
        pc_plus_4;

endmodule



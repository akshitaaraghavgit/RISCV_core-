module riscv_core (
    input  wire clk,
    input  wire reset
);

    //========================================================
    // PC
    //========================================================
    reg [31:0] pc;

    //========================================================
    // Instruction Memory
    // 256 words = 1 KB
    //========================================================
    reg [31:0] imem [0:255];

    wire [31:0] instruction;
    assign instruction = imem[pc[9:2]];

    //========================================================
    // Register File
    // 32 registers, each 32-bit
    //========================================================
    reg [31:0] regs [0:31];

    wire [4:0] rs1;
    wire [4:0] rs2;
    wire [4:0] rd;

    assign rs1 = instruction[19:15];
    assign rs2 = instruction[24:20];
    assign rd  = instruction[11:7];

    wire [31:0] rs1_data;
    wire [31:0] rs2_data;

    assign rs1_data = (rs1 == 0) ? 32'b0 : regs[rs1];
    assign rs2_data = (rs2 == 0) ? 32'b0 : regs[rs2];

    //========================================================
    // Instruction fields
    //========================================================
    wire [6:0] opcode;
    wire [2:0] funct3;
    wire [6:0] funct7;

    assign opcode = instruction[6:0];
    assign funct3 = instruction[14:12];
    assign funct7 = instruction[31:25];

    //========================================================
    // Immediate Generator
    //========================================================

    wire [31:0] imm_i;
    wire [31:0] imm_s;
    wire [31:0] imm_b;

    // I-type immediate
    assign imm_i = {{20{instruction[31]}},
                    instruction[31:20]};

    // S-type immediate
    assign imm_s = {{20{instruction[31]}},
                    instruction[31:25],
                    instruction[11:7]};

    // B-type immediate
    assign imm_b = {{19{instruction[31]}},
                    instruction[31],
                    instruction[7],
                    instruction[30:25],
                    instruction[11:8],
                    1'b0};

    //========================================================
    // Control Signals
    //========================================================

    reg reg_write;
    reg mem_read;
    reg mem_write;
    reg alu_src;
    reg mem_to_reg;
    reg branch;

    reg [3:0] alu_control;

    localparam ALU_ADD = 4'b0000;
    localparam ALU_SUB = 4'b0001;
    localparam ALU_AND = 4'b0010;
    localparam ALU_OR  = 4'b0011;
    localparam ALU_XOR = 4'b0100;

    //========================================================
    // Control Unit
    //========================================================

    always @(*) begin

        // Default values
        reg_write  = 0;
        mem_read   = 0;
        mem_write  = 0;
        alu_src    = 0;
        mem_to_reg = 0;
        branch     = 0;
        alu_control = ALU_ADD;

        case (opcode)

            // R-type
            7'b0110011: begin

                reg_write = 1;
                alu_src   = 0;

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
                alu_src   = 1;
                alu_control = ALU_ADD;
            end

            // LW
            7'b0000011: begin
                reg_write  = 1;
                mem_read   = 1;
                alu_src    = 1;
                mem_to_reg = 1;
                alu_control = ALU_ADD;
            end

            // SW
            7'b0100011: begin
                mem_write = 1;
                alu_src   = 1;
                alu_control = ALU_ADD;
            end

            // BEQ
            7'b1100011: begin
                branch = 1;
                alu_src = 0;
                alu_control = ALU_SUB;
            end

            default: begin
                // Do nothing for unsupported instruction
            end

        endcase
    end

    //========================================================
    // ALU B MUX
    // Register OR Immediate
    //========================================================

    wire [31:0] alu_b;

    assign alu_b = alu_src ? 
                   ((opcode == 7'b0100011) ? imm_s : imm_i) :
                   rs2_data;

    //========================================================
    // ALU
    //========================================================

    reg [31:0] alu_result;

    always @(*) begin

        case (alu_control)

            ALU_ADD:
                alu_result = rs1_data + alu_b;

            ALU_SUB:
                alu_result = rs1_data - alu_b;

            ALU_AND:
                alu_result = rs1_data & alu_b;

            ALU_OR:
                alu_result = rs1_data | alu_b;

            ALU_XOR:
                alu_result = rs1_data ^ alu_b;

            default:
                alu_result = 32'b0;

        endcase
    end

    //========================================================
    // Zero signal for BEQ
    //========================================================

    wire alu_zero;

    assign alu_zero = (alu_result == 32'b0);

    //========================================================
    // Data Memory
    //========================================================

    reg [31:0] dmem [0:255];

    wire [31:0] memory_data;

    assign memory_data = dmem[alu_result[9:2]];

    //========================================================
    // Write-back MUX
    // ALU result OR Memory data
    //========================================================

    wire [31:0] writeback_data;

    assign writeback_data =
        mem_to_reg ? memory_data : alu_result;

    //========================================================
    // Next PC MUX
    // PC+4 OR Branch Target
    //========================================================

    wire [31:0] pc_plus_4;
    wire [31:0] branch_target;
    wire [31:0] next_pc;

    assign pc_plus_4 = pc + 32'd4;

    assign branch_target = pc + imm_b;

    assign next_pc =
        (branch && alu_zero) ?
        branch_target :
        pc_plus_4;

    //========================================================
    // Sequential Logic
    //========================================================

    integer i;

    always @(posedge clk or posedge reset) begin

        if (reset) begin

            pc <= 32'b0;

            for (i = 0; i < 32; i = i + 1)
                regs[i] <= 32'b0;

        end

        else begin

            // PC update
            pc <= next_pc;

            // Register write-back
            if (reg_write && rd != 0)
                regs[rd] <= writeback_data;

            // x0 is ALWAYS zero
            regs[0] <= 32'b0;

            // Store
            if (mem_write)
                dmem[alu_result[9:2]] <= rs2_data;

        end

    end

endmodule




`timescale 1ns/1ps

module tb;

    reg clk;
    reg reset;

    // Create processor
    riscv_core dut (
        .clk(clk),
        .reset(reset)
    );

    // Clock
    always #5 clk = ~clk;

    initial begin

        clk = 0;
        reset = 1;

        //====================================================
        // Program
        //====================================================

        // x1 = 5
        // addi x1, x0, 5
        dut.imem[0] = 32'h00500093;

        // x2 = 8
        // addi x2, x0, 8
        dut.imem[1] = 32'h00800113;

        // x3 = x1 + x2
        // add x3, x1, x2
        dut.imem[2] = 32'h002081B3;

        // memory[0] = x3
        // sw x3, 0(x0)
        dut.imem[3] = 32'h00302023;

        // x4 = memory[0]
        // lw x4, 0(x0)
        dut.imem[4] = 32'h00002203;

        // if x3 == x4, skip next instruction
        // beq x3, x4, +8
        dut.imem[5] = 32'h00418463;

        // This instruction should be skipped
        // x5 = 99
        dut.imem[6] = 32'h06300293;

        // x5 = 42
        dut.imem[7] = 32'h02A00293;

        // Release reset
        #12;
        reset = 0;

        // Let processor execute
        repeat (10)
            @(posedge clk);

        //====================================================
        // Display results
        //====================================================

        $display("--------------------------------");
        $display("RISC-V PROCESSOR RESULTS");
        $display("--------------------------------");

        $display("x1 = %0d", dut.regs[1]);
        $display("x2 = %0d", dut.regs[2]);
        $display("x3 = %0d", dut.regs[3]);
        $display("x4 = %0d", dut.regs[4]);
        $display("x5 = %0d", dut.regs[5]);

        $display("Memory[0] = %0d", dut.dmem[0]);

        //====================================================
        // Checks
        //====================================================

        if (dut.regs[1] == 5)
            $display("PASS: x1 = 5");
        else
            $display("FAIL: x1");

        if (dut.regs[2] == 8)
            $display("PASS: x2 = 8");
        else
            $display("FAIL: x2");

        if (dut.regs[3] == 13)
            $display("PASS: ADD worked");
        else
            $display("FAIL: ADD");

        if (dut.regs[4] == 13)
            $display("PASS: LW worked");
        else
            $display("FAIL: LW");

        if (dut.dmem[0] == 13)
            $display("PASS: SW worked");
        else
            $display("FAIL: SW");

        if (dut.regs[5] == 42)
            $display("PASS: BEQ worked");
        else
            $display("FAIL: BEQ");

        $display("--------------------------------");

        $finish;

    end

endmodule

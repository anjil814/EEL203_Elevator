module elevator_top(
    input  wire        clk,
    input  wire        rst_n,
    input  wire [3:0]  call_up,
    input  wire [3:0]  call_down,
    input  wire [3:0]  cab_req,
    output reg  [1:0]  cur_floor,
    output reg         moving,
    output reg         dir_up,
    output reg         door_open
);

    localparam IDLE       = 2'b00;
    localparam MOVING     = 2'b01;
    localparam DOOR_OPEN  = 2'b10;

    parameter integer DOOR_OPEN_CYCLES = 50;

    reg [1:0] state, next_state;
    reg [7:0] door_timer;

    wire [3:0] any_request;
    assign any_request = cab_req | call_up | call_down;

    function automatic has_request_above(input [1:0] f);
        integer i;
        begin
            has_request_above = 0;
            for (i = f+1; i <= 3; i = i + 1) begin
                if (any_request[i]) has_request_above = 1;
            end
        end
    endfunction

    function automatic has_request_below(input [1:0] f);
        integer i;
        begin
            has_request_below = 0;
            for (i = 0; i <= f-1; i = i + 1) begin
                if (any_request[i]) has_request_below = 1;
            end
        end
    endfunction

    wire stop_here;
    assign stop_here = any_request[cur_floor];

    always @(posedge clk or negedge rst_n) begin
        if (!rst_n) begin
            state      <= IDLE;
            cur_floor  <= 2'd0;
            moving     <= 1'b0;
            dir_up     <= 1'b1;
            door_open  <= 1'b0;
            door_timer <= 0;
        end else begin
            state <= next_state;

            case (state)
                IDLE: begin
                    moving    <= 1'b0;
                    door_open <= 1'b0;
                    door_timer <= 0;
                end

                MOVING: begin
                    moving <= 1'b1;
                    if (dir_up) begin
                        if (cur_floor != 2'd3) cur_floor <= cur_floor + 1;
                    end else begin
                        if (cur_floor != 2'd0) cur_floor <= cur_floor - 1;
                    end
                end

                DOOR_OPEN: begin
                    moving <= 1'b0;
                    door_open <= 1'b1;
                    if (door_timer < DOOR_OPEN_CYCLES) begin
                        door_timer <= door_timer + 1;
                    end
                end

                default: begin
                    moving <= 1'b0;
                end
            endcase
        end
    end

    always @* begin
        next_state = state;

        case (state)
            IDLE: begin
                if (any_request) begin
                    if (has_request_above(cur_floor)) begin
                        dir_up = 1'b1;
                    end else if (has_request_below(cur_floor)) begin
                        dir_up = 1'b0;
                    end
                    next_state = MOVING;
                end else next_state = IDLE;
            end

            MOVING: begin
                if (stop_here) begin
                    next_state = DOOR_OPEN;
                end else begin
                    if (dir_up) begin
                        if (has_request_above(cur_floor)) next_state = MOVING;
                        else if (has_request_below(cur_floor)) begin
                            dir_up = 1'b0;
                            next_state = MOVING;
                        end else next_state = IDLE;
                    end else begin
                        if (has_request_below(cur_floor)) next_state = MOVING;
                        else if (has_request_above(cur_floor)) begin
                            dir_up = 1'b1;
                            next_state = MOVING;
                        end else next_state = IDLE;
                    end
                end
            end

            DOOR_OPEN: begin
                if (door_timer >= DOOR_OPEN_CYCLES) begin
                    next_state = IDLE;
                end else next_state = DOOR_OPEN;
            end

            default: next_state = IDLE;
        endcase
    end

endmodule


module tb_elevator;
    reg clk;
    reg rst_n;
    reg [3:0] call_up;
    reg [3:0] call_down;
    reg [3:0] cab_req;
    wire [1:0] cur_floor;
    wire moving;
    wire dir_up;
    wire door_open;

    elevator_top uut(
        .clk(clk),
        .rst_n(rst_n),
        .call_up(call_up),
        .call_down(call_down),
        .cab_req(cab_req),
        .cur_floor(cur_floor),
        .moving(moving),
        .dir_up(dir_up),
        .door_open(door_open)
    );

    initial begin
        clk = 0;
        forever #5 clk = ~clk;
    end

    initial begin
        rst_n = 0;
        call_up = 4'b0000;
        call_down = 4'b0000;
        cab_req = 4'b0000;
        #20;
        rst_n = 1;

        #20;
        call_down = 4'b0100;
        #50;
        call_down = 4'b0000;

        #80;
        cab_req = 4'b0001;
        #40;
        cab_req = 4'b0000;

        #100;
        call_up = 4'b1000;
        #40;
        call_up = 4'b0000;

        #400;
        $finish;
    end

    initial begin
        $dumpfile("elevator_tb.vcd");
        $dumpvars(0, tb_elevator);
    end

endmodule

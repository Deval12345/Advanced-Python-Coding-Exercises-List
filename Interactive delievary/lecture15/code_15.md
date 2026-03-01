# Capstone: no single code block; students implement the pipeline.
# Optional skeleton (not keyed to CONCEPT/EXAMPLE):

# class Record:
#     __slots__ = ("id", "value", "timestamp")
#     ...
#
# def read_records(path):
#     with open(path) as f:
#         for line in f:
#             yield parse_line(line)
#
# with PipelineSession() as session:
#     for record in read_records("data.csv"):
#         try:
#             if validator(record): session.write(record)
#         except ValueError: ...

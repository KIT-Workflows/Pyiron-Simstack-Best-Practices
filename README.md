# Minimum implementation in SimStack and pyiron

## What is a typical *classical* software package:

| Names | Input files | Executable | Output files|
|--|--|--|--|
|VASP| INCAR, POSCAR, POTCAR etc. | `vasp` | OUTCAR |
| Gnuplot | xyz-data files, script files | `gnuplot "filename"` | fit.log |
| LAMMPS | lmp.in, structure.xyz | `lammps < lmp.in` | log.lammps |
| phonopy | mesh.conf, phonopy_disp.yaml | `phonopy -p mesh.conf` | phonopy.yaml|

## What does the user have to provide:

- Input generator (parameters -> input files)
- Executable
- Output parsers

## Minimum example with Gnuplot

### Input generator

```python!
def generate_input(x, y, path):
    data_file = os.path.join(path, 'xy.dat')
    np.savetxt(data_file, np.stack((x, y), axis=-1))
    gnuplot_script = 
    "f(x) = a*x+b\nfit f(x) \"xy.dat\" using 1:2 via a,b"
    gnuplot_file = os.path.join(path, 'input.gnu')
    with open(gnuplot_file, "w") as f:
        f.write(gnuplot_script)
```

### Executable
```bash
gnuplot "input.gnu"
```

### Output parser

```python
def _get_gnuplot_parameter_line(content):
    for ii, line in enumerate(content):
        if line.startswith("Final set of parameters"):
            return ii + 2
    raise ValueError("line not found")
    
def get_gnuplot_parameters(content):
    parameter_line = _get_gnuplot_parameter_line(content)
    slope = float(content[parameter_line].split("=")[1].split()[0])
    intercept = float(content[parameter_line + 1].split("=")[1].split()[0])
    return slope, intercept

def parse_output(path):
    data_file = os.path.join(path, 'fit.log')
    with open(data_file, "r") as f:
        content = f.readlines()
    return get_gnuplot_parameters(content)
```

## pyiron

### Node declaration

```python
class Gnuplot(TemplateJob):
    def __init__(self, project, job_name):
        super().__init__(project, job_name)
        self.executable = "gnuplot \"input.gnu\"" # executable, required
        # self.input is available to the user and it can store anything that can be serialized.
        # The content will be automatically stored when `job.run()` is called
        self.input.x = None
        self.input.y = None

    # `write_input` is called before the executable is called
    def write_input(self):
        generate_input(self.input.x, self.input.y, self.working_directory)

    # `collect_output` is called after the executable is called.
    # It should include the parsing process
    def collect_output(self):
        # Just like `self.input`, `self.output` is available to the user.
        # The content will be stored when `self.to_hdf()` is called.
        self.output.slope, self.output.intercept = parse_output(self.working_directory)
        self.to_hdf()
        
    # Optional function. If defined, it will be called before `write_input`
    def validate_ready_to_run(self):
        if self.input.x is None or self.input.y is None:
            raise ValueError("x and/or y is not defined")
```

### User interface

```python
job = pr.create_job(Gnuplot, job_name="gnuplot_example")
job.input.x = x
job.input.y = y
job.run()
```

## SimStack


### Input generator (input_generator.py)

```python
import os, yaml
import numpy as np

def read_xy_from_file(filename):
    x_values = []
    y_values = []
    with open(filename, 'r') as file:
        for line in file:
            x_str, y_str = line.split()
            x_values.append(float(x_str))
            y_values.append(float(y_str))
    return x_values, y_values

def create_xy_lists(data_dict):
    x_values, y_values = [], []
    xy_lists = data_dict.get("x and y lists", [])    
    for item in xy_lists:
        x_values.append(item.get('x'))
        y_values.append(item.get('y'))    
    return x_values, y_values

def generate_input(x, y, working_directory):
    data_file = os.path.join(working_directory, 'xy.dat')
    np.savetxt(data_file, np.stack((x, y), axis=-1))
    gnuplot_script = "f(x) = a*x+b\nfit f(x) \"xy.dat\" using 1:2 via a,b"
    gnuplot_file = os.path.join(working_directory, 'input.gnu')
    with open(gnuplot_file, "w") as f:
        f.write(gnuplot_script)
    return gnuplot_file

current_directory = os.getcwd()
with open("rendered_wano.yml") as file:
    wano_file = yaml.full_load(file) 

if wano_file["Upload file"]:
    x,y = read_xy_from_file("file.dat")
else:
    x,y = create_xy_lists(wano_file)

generate_input(x,y, current_directory)
```
### Output Parser (output_parser.py)

```python
import os

def _get_gnuplot_parameter_line(content):
    for ii, line in enumerate(content):
        if line.startswith("Final set of parameters"):
            return ii + 2
    raise ValueError("line not found")

def get_gnuplot_parameters(content):
    parameter_line = _get_gnuplot_parameter_line(content)
    slope = float(content[parameter_line].split("=")[1].split()[0])
    intercept = float(content[parameter_line + 1].split("=")[1].split()[0])
    return slope, intercept

def parse_output(working_directory):
    data_file = os.path.join(working_directory, 'fit.log')
    with open(data_file, "r") as f:
        content = f.readlines()
    return get_gnuplot_parameters(content)

if __name__ == "__main__":
    current_directory = os.getcwd()
    print(parse_output(current_directory))
```




### WaNo 

```xml!
<WaNoTemplate>
  <WaNoMeta>
    <Description>
      This WaNo generate a fit.log file using Gnuplot.
    </Description>
    <Keyword>Gnuplot</Keyword>
</WaNoMeta>

<WaNoRoot name="Gnuplot">
  <WaNoBool name="Upload file">False</WaNoBool> 
  <WaNoFile visibility_condition="%s == True" visibility_var_path="Upload file" logical_filename="file.dat" name="file dat">path to file1</WaNoFile>

  <WaNoMultipleOf visibility_condition="%s == False" visibility_var_path="Upload file"  name="x and y lists">
    <Element id="0">          
      <WaNoFloat name="x" description = "X coodinate">0.48</WaNoFloat>
      <WaNoFloat name="y" description = "Y coodinate">1.8</WaNoFloat>
    </Element>
  </WaNoMultipleOf>
</WaNoRoot>

<WaNoExecCommand>
  python input_generator.py
  gnuplot "input.gnu"
  python output_parser.py
</WaNoExecCommand>

<WaNoInputFiles>
  <WaNoInputFile logical_filename="input_generator.py">input_generator.py</WaNoInputFile>
  <WaNoInputFile logical_filename="output_parser.py">output_parser.py</WaNoInputFile>
</WaNoInputFiles>

<WaNoOutputFiles>
  <WaNoOutputFile>fit.log</WaNoOutputFile>
</WaNoOutputFiles>
</WaNoTemplate>
```

### User interface

![Screenshot from 2024-02-29 17-21-06](https://hackmd.io/_uploads/rkE9ZYJT6.png)
![Screenshot from 2024-02-29 17-20-55](https://hackmd.io/_uploads/S149-YkTa.png)

### Input generator

```python!
def generate_input(x, y, path):
    data_file = os.path.join(path, 'xy.dat')
    np.savetxt(data_file, np.stack((x, y), axis=-1))
    gnuplot_script = 
    "f(x) = a*x+b\nfit f(x) \"xy.dat\" using 1:2 via a,b"
    gnuplot_file = os.path.join(path, 'input.gnu')
    with open(gnuplot_file, "w") as f:
        f.write(gnuplot_script)

        
        
gnuplot "input.gnu"



def _get_gnuplot_parameter_line(content):
    for ii, line in enumerate(content):
        if line.startswith("Final set of parameters"):
            return ii + 2
    raise ValueError("line not found")
    
def get_gnuplot_parameters(content):
    parameter_line = _get_gnuplot_parameter_line(content)
    slope = float(content[parameter_line].split("=")[1].split()[0])
    intercept = float(content[parameter_line + 1].split("=")[1].split()[0])
    return slope, intercept

def parse_output(path):
    data_file = os.path.join(path, 'fit.log')
    with open(data_file, "r") as f:
        content = f.readlines()
    return get_gnuplot_parameters(content)
```

+++
title = "AIMET QUANTSIM [1]"
date = 2024-04-27T21:03:41+08:00
draft = false
+++


# 量化基础

# QuantSim

# 代码解析

QuantizationSimModel
``` python {.1}
class QuantizationSimModel:
    def __init__(self, 
                 model: torch.nn.Module, 
                 dummy_input: Union[torch.Tensor, Tuple],
                 quant_scheme: Union[str, QuantScheme] = QuantScheme.post_training_tf_enhanced,
                 rounding_mode: str = 'nearest', 
                 default_output_bw: int = 8, 
                 default_param_bw: int = 8,
                 in_place: bool = False, 
                 config_file: str = None,
                 default_data_type: QuantizationDataType = QuantizationDataType.int):
        """
        Constructor for QuantizationSimModel.

        :param model: Model to add simulation ops to
        :param dummy_input: Dummy input to the model. Used to parse model graph. If the model has more than one input,
                            pass a tuple. User is expected to place the tensors on the appropriate device.
        :param quant_scheme: Quantization scheme. The Quantization scheme is used to compute the Quantization encodings.
                             There are multiple schemes available. Please refer the QuantScheme enum definition.
        :param rounding_mode: Rounding mode. Supported options are 'nearest' or 'stochastic'
        :param default_output_bw: Default bitwidth (4-31) to use for quantizing all layer inputs and outputs
        :param default_param_bw: Default bitwidth (4-31) to use for quantizing all layer parameters
        :param in_place: If True, then the given 'model' is modified in-place to add quant-sim nodes.
                Only suggested use of this option is when the user wants to avoid creating a copy of the model
        :param config_file: Path to Configuration file for model quantizers
        :param default_data_type: Default data type to use for quantizing all layer inputs, outputs and parameters.
                                 Possible options are QuantizationDataType.int and QuantizationDataType.float.
                                 Note that the mode default_data_type=QuantizationDataType.float is only supported with
                                 default_output_bw=16 and default_param_bw=16
        """
        # Perform sanity checks on inputs
        validate_quantsim_inputs(quant_scheme, rounding_mode, default_output_bw, default_param_bw,
                                 default_data_type)
        # save some parameters
        if in_place:
            self.model = model
        else:
            self.model = copy.deepcopy(model)

        try:
            self.connected_graph = ConnectedGraph(self.model, dummy_input)
        except (torch.jit.TracingCheckError, AssertionError):
            self.connected_graph = None

        if isinstance(quant_scheme, str):
            if quant_scheme == 'tf':
                quant_scheme = QuantScheme.post_training_tf
            elif quant_scheme == 'tf_enhanced':
                quant_scheme = QuantScheme.post_training_tf_enhanced
            elif quant_scheme == 'percentile':
                quant_scheme = QuantScheme.post_training_percentile
        self._quant_scheme = quant_scheme
        self._rounding_mode = rounding_mode
        self._default_output_bw = default_output_bw
        self._default_param_bw = default_param_bw
        self._is_conditional = False
        self._module_marker_map = {}
        self._percentile_value = 100 # default percentile value
        self._excluded_layer_names = []

        # Add quantization layers
        num_inout_tensors = utils.find_num_inout_tensors_per_module(self.model, dummy_input)

        self._add_quantization_wrappers(self.model, num_inout_tensors, default_data_type)
        self._set_tensor_quantizers_for_consts()

        # Disable bias quantization
        self.exclude_param_from_quantization("bias")

        # override specific quantizers to tf mode in transformer model
        self._override_quant_config_for_transformer_layers()

        quantsim_configurator = self.configure_quantization_ops(config_file, default_output_bw, default_param_bw,
                                                                default_data_type)

        self.quant_args = extract_global_quantizer_args(quant_scheme, quantsim_configurator)

        # pylint: disable=protected-access
        self._hw_version = quantsim_configurator._get_hw_version()
        self._supported_kernels = quantsim_configurator.get_supported_kernels()
        self._validate_supported_kernels_for_quantizers(SUPPORTED_KERNELS_ACTION)
```
